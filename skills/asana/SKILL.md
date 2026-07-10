# asana

<!-- GUILDOS-SKILL name="asana" -->

<!-- GUILDOS-DESC -->
Read tasks and projects from Asana workspaces. Read all incomplete tasks from a project (by name or ID) with titles, notes, sections, assignees, and completion status, and read recent comments/stories on a task. The full CRUD connector (search/read/write/delete tasks, projects, sections, comments) is the folded-in asana weapon over the Asana REST API v1, using ASANA_ACCESS_TOKEN.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/asana/SKILL.md -->
```markdown
---
name: asana
description: >-
  Read tasks and projects from Asana workspaces. Read all incomplete tasks from
  a project (by name or ID) with titles, notes, sections, assignees, and
  completion status, and read recent comments/stories on a task. The full CRUD
  connector (search/read/write/delete tasks, projects, sections, comments) is
  the folded-in asana weapon over the Asana REST API v1, using
  ASANA_ACCESS_TOKEN.
---

# Asana

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/asana/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto asana <action>` (or read the source directly).

## Actions

- **readProjectTasks** — Read all incomplete tasks from an Asana project by name or ID. Returns task titles, notes, sections, assignees, and completion status.
- **readTaskComments** — Read recent comments/stories on an Asana task.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/asana/index.js -->
```js
/**
 * Asana skill book — read-only access to Asana projects and tasks.
 * Uses ASANA_ACCESS_TOKEN from environment.
 */
import { skillActionOk, skillActionErr } from "@/libs/skill_book/actionResult.js";

const ASANA_BASE = "https://app.asana.com/api/1.0";

function asanaHeaders() {
  const token = process.env.ASANA_ACCESS_TOKEN;
  if (!token) throw new Error("ASANA_ACCESS_TOKEN not set in environment.");
  return { Authorization: `Bearer ${token}`, "Content-Type": "application/json" };
}

async function asanaGet(path, params = {}) {
  const url = new URL(`${ASANA_BASE}${path}`);
  for (const [k, v] of Object.entries(params)) {
    if (v != null) url.searchParams.set(k, String(v));
  }
  const res = await fetch(url.toString(), { headers: asanaHeaders() });
  const json = await res.json().catch(() => ({}));
  if (!res.ok) return { ok: false, error: json.errors?.[0]?.message || res.statusText, data: null };
  return { ok: true, data: json.data, next_page: json.next_page };
}

export const skillBook = {
  id: "asana",
  title: "Asana",
  description: "Read tasks and projects from Asana workspaces.",
  steps: [],
  toc: {
    readProjectTasks: {
      description: "Read all incomplete tasks from an Asana project by name or ID. Returns task titles, notes, sections, assignees, and completion status.",
      input: {
        project_name: "string — project name to search for (e.g. '[CJ] backlogs'). Used if project_id is not provided.",
        project_id: "string — Asana project GID. If provided, project_name is ignored.",
        include_completed: "boolean — include completed tasks (default: false)",
        workspace_id: "string — workspace GID (default: searches all workspaces)",
      },
      output: {
        tasks: "array of { gid, name, notes, completed, assignee, section, tags }",
        project: "object with { gid, name } of the matched project",
      },
    },
    readTaskComments: {
      description: "Read recent comments/stories on an Asana task.",
      input: {
        task_id: "string — Asana task GID",
        limit: "number — max comments to return (default: 10)",
      },
      output: {
        comments: "array of { author, text, created_at, type }",
      },
    },
  },
};

/**
 * Find a project by name across workspaces.
 */
async function resolveProjectId(projectName, workspaceId) {
  if (workspaceId) {
    const { ok, data } = await asanaGet("/projects", {
      workspace: workspaceId,
      opt_fields: "name",
      limit: 100,
    });
    if (ok && data) {
      const match = data.find((p) => p.name.toLowerCase().includes(projectName.toLowerCase()));
      if (match) return { ok: true, project: match };
    }
  }

  // Search all workspaces
  const { ok: wOk, data: workspaces } = await asanaGet("/workspaces", { limit: 100 });
  if (!wOk || !workspaces) return { ok: false, error: "Failed to list workspaces" };

  for (const ws of workspaces) {
    const { ok, data } = await asanaGet("/projects", {
      workspace: ws.gid,
      opt_fields: "name",
      limit: 100,
    });
    if (ok && data) {
      const match = data.find((p) => p.name.toLowerCase().includes(projectName.toLowerCase()));
      if (match) return { ok: true, project: match };
    }
  }
  return { ok: false, error: `Project "${projectName}" not found in any workspace.` };
}

export async function readProjectTasks(_userId, input) {
  const inObj = typeof input === "object" && input ? input : {};
  let projectId = inObj.project_id || null;
  let projectInfo = null;
  const includeCompleted = inObj.include_completed === true || inObj.include_completed === "true";

  if (!projectId) {
    const name = String(inObj.project_name || "").trim();
    if (!name) return skillActionErr("Either project_name or project_id is required.");
    const resolved = await resolveProjectId(name, inObj.workspace_id);
    if (!resolved.ok) return skillActionErr(resolved.error);
    projectId = resolved.project.gid;
    projectInfo = resolved.project;
  }

  // Fetch tasks with pagination
  const allTasks = [];
  let offset = null;
  const optFields = "name,notes,completed,assignee.name,memberships.section.name,tags.name,created_at,modified_at";

  do {
    const params = { opt_fields: optFields, limit: 100 };
    if (!includeCompleted) params.completed_since = "now"; // only incomplete
    if (offset) params.offset = offset;

    const { ok, data, next_page, error } = await asanaGet(`/projects/${projectId}/tasks`, params);
    if (!ok) return skillActionErr(`Failed to fetch tasks: ${error}`);
    if (data) allTasks.push(...data);
    offset = next_page?.offset || null;
  } while (offset);

  // Normalize
  const tasks = allTasks.map((t) => ({
    gid: t.gid,
    name: t.name,
    notes: (t.notes || "").slice(0, 500),
    completed: t.completed,
    assignee: t.assignee?.name || null,
    section: t.memberships?.[0]?.section?.name || null,
    tags: (t.tags || []).map((tag) => tag.name),
    modified_at: t.modified_at,
  }));

  if (!projectInfo && projectId) {
    const { data } = await asanaGet(`/projects/${projectId}`, { opt_fields: "name" });
    projectInfo = data ? { gid: data.gid, name: data.name } : { gid: projectId, name: "unknown" };
  }

  return skillActionOk({
    items: {
      tasks: JSON.stringify(tasks),
      project: JSON.stringify(projectInfo),
    },
    msg: `Fetched ${tasks.length} tasks from ${projectInfo?.name || projectId}`,
  });
}

export async function readTaskComments(_userId, input) {
  const inObj = typeof input === "object" && input ? input : {};
  const taskId = String(inObj.task_id || "").trim();
  if (!taskId) return skillActionErr("task_id is required.");
  const limit = Math.min(Number(inObj.limit) || 10, 50);

  const { ok, data, error } = await asanaGet(`/tasks/${taskId}/stories`, {
    opt_fields: "created_by.name,text,created_at,type,resource_subtype",
    limit,
  });
  if (!ok) return skillActionErr(`Failed to fetch comments: ${error}`);

  const comments = (data || [])
    .filter((s) => s.resource_subtype === "comment_added")
    .map((s) => ({
      author: s.created_by?.name || "unknown",
      text: (s.text || "").slice(0, 500),
      created_at: s.created_at,
      type: s.resource_subtype,
    }));

  return skillActionOk({
    items: { comments: JSON.stringify(comments) },
    msg: `Fetched ${comments.length} comments for task ${taskId}`,
  });
}

export default { skillBook, readProjectTasks, readTaskComments };
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/asana/index.js -->
```js
/**
 * Asana weapon — unified Asana REST API connector.
 *
 * Auth: ASANA_ACCESS_TOKEN from profiles.env_vars or process.env.
 */

export const toc = {
  searchWorkspaces: "Search visible Asana workspaces for the authenticated user.",
  searchProjects: "Search Asana projects by name or workspace.",
  readProject: "Read a single Asana project by id.",
  writeProject: "Update an Asana project's name and/or notes (description).",
  searchTasks: "Search Asana tasks across projects with filters.",
  readTask: "Read full details of one Asana task.",
  writeTask: "Create or update an Asana task (upsert).",
  deleteTask: "Delete an Asana task.",
  readSections: "Read sections of an Asana project.",
  readComments: "Read stories/comments on an Asana task.",
  writeComment: "Post a comment on an Asana task.",
  editComment: "Edit the text of an existing comment story (only the author can edit).",
  deleteComment: "Delete a comment story by its GID (only the author can delete).",
};
import { database } from "@/libs/council/database";

const API_BASE = "https://app.asana.com/api/1.0";

// ---------------------------------------------------------------------------
// Auth
// ---------------------------------------------------------------------------

async function getToken(userId) {
  if (process.env.ASANA_ACCESS_TOKEN) return process.env.ASANA_ACCESS_TOKEN;
  if (!userId) return null;
  const db = await database.init("service");
  const { data } = await db
    .from("profiles")
    .select("env_vars")
    .eq("id", userId)
    .single();
  return data?.env_vars?.ASANA_ACCESS_TOKEN ?? null;
}

async function asanaFetch(path, opts = {}, userId) {
  const token = await getToken(userId);
  if (!token) throw new Error("Missing ASANA_ACCESS_TOKEN");
  const url = path.startsWith("http") ? path : `${API_BASE}${path}`;
  const res = await fetch(url, {
    ...opts,
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json",
      ...opts.headers,
    },
  });
  if (!res.ok) {
    const body = await res.text();
    throw new Error(`Asana API ${res.status}: ${body.slice(0, 500)}`);
  }
  const json = await res.json();
  return json.data !== undefined ? json : json;
}

// ---------------------------------------------------------------------------
// Workspaces
// ---------------------------------------------------------------------------

/**
 * List all workspaces the token has access to.
 * @param {object} [input]
 * @param {string} [userId]
 */
export async function searchWorkspaces(input = {}, userId) {
  const res = await asanaFetch("/workspaces?limit=100", {}, userId);
  return { workspaces: res.data || [] };
}

// ---------------------------------------------------------------------------
// Projects
// ---------------------------------------------------------------------------

/**
 * Search projects by name or list all in a workspace.
 * @param {{ workspace_id?: string, query?: string, limit?: number }} input
 * @param {string} [userId]
 */
export async function searchProjects({ workspace_id, query, limit = 100 } = {}, userId) {
  if (workspace_id) {
    const res = await asanaFetch(
      `/projects?workspace=${workspace_id}&opt_fields=name,archived,color&limit=${limit}`,
      {},
      userId,
    );
    let projects = res.data || [];
    if (query) {
      const q = query.toLowerCase();
      projects = projects.filter((p) => p.name.toLowerCase().includes(q));
    }
    return { projects };
  }
  // Search all workspaces
  const { workspaces } = await searchWorkspaces({}, userId);
  const all = [];
  for (const ws of workspaces) {
    const res = await asanaFetch(
      `/projects?workspace=${ws.gid}&opt_fields=name,archived,color&limit=${limit}`,
      {},
      userId,
    );
    let projects = res.data || [];
    if (query) {
      const q = query.toLowerCase();
      projects = projects.filter((p) => p.name.toLowerCase().includes(q));
    }
    all.push(...projects);
  }
  return { projects: all };
}

/**
 * Read a single project by GID.
 * @param {{ project_id: string }} input
 * @param {string} [userId]
 */
export async function readProject({ project_id } = {}, userId) {
  if (!project_id) throw new Error("project_id is required");
  const res = await asanaFetch(
    `/projects/${project_id}?opt_fields=name,notes,archived,color,created_at,modified_at,owner.name,members.name`,
    {},
    userId,
  );
  return res.data || res;
}

/**
 * Update an Asana project (name and/or notes/description).
 * @param {{ project_id: string, name?: string, notes?: string }} input
 * @param {string} [userId]
 */
export async function writeProject({ project_id, name, notes } = {}, userId) {
  if (!project_id) throw new Error("project_id is required");
  const body = {};
  if (name !== undefined) body.name = name;
  if (notes !== undefined) body.notes = notes;
  if (!Object.keys(body).length) throw new Error("Nothing to update — provide name and/or notes");
  const res = await asanaFetch(
    `/projects/${project_id}`,
    { method: "PUT", body: JSON.stringify({ data: body }) },
    userId,
  );
  return res.data || res;
}

// ---------------------------------------------------------------------------
// Tasks
// ---------------------------------------------------------------------------

/**
 * Search tasks in a project or by assignee.
 * @param {{ project_id?: string, assignee?: string, workspace_id?: string, completed?: boolean, limit?: number }} input
 * @param {string} [userId]
 */
export async function searchTasks({ project_id, assignee, workspace_id, completed, limit = 100 } = {}, userId) {
  const optFields = "name,notes,completed,assignee.name,memberships.section.name,tags.name,due_on,created_at,modified_at";

  if (project_id) {
    const params = new URLSearchParams({ opt_fields: optFields, limit: String(limit) });
    if (completed === false) params.set("completed_since", "now");
    const allTasks = [];
    let offset = null;
    do {
      if (offset) params.set("offset", offset);
      const res = await asanaFetch(`/projects/${project_id}/tasks?${params}`, {}, userId);
      if (res.data) allTasks.push(...res.data);
      offset = res.next_page?.offset || null;
    } while (offset && allTasks.length < limit);
    return { tasks: allTasks.slice(0, limit) };
  }

  if (assignee && workspace_id) {
    const params = new URLSearchParams({
      assignee,
      workspace: workspace_id,
      opt_fields: optFields,
      limit: String(limit),
    });
    if (completed === false) params.set("completed_since", "now");
    const res = await asanaFetch(`/tasks?${params}`, {}, userId);
    return { tasks: res.data || [] };
  }

  throw new Error("Either project_id or both assignee + workspace_id are required.");
}

/**
 * Read a single task by GID.
 * @param {{ task_id: string }} input
 * @param {string} [userId]
 */
export async function readTask({ task_id } = {}, userId) {
  if (!task_id) throw new Error("task_id is required");
  const res = await asanaFetch(
    `/tasks/${task_id}?opt_fields=name,notes,completed,assignee.name,memberships.section.name,tags.name,due_on,created_at,modified_at,parent.name,custom_fields`,
    {},
    userId,
  );
  return res.data || res;
}

/**
 * Create or update a task.
 * @param {{ task_id?: string, project_id?: string, name?: string, notes?: string, assignee?: string, due_on?: string, completed?: boolean }} input
 * @param {string} [userId]
 */
export async function writeTask({ task_id, project_id, name, notes, assignee, due_on, completed } = {}, userId) {
  const body = {};
  if (name != null) body.name = name;
  if (notes != null) body.notes = notes;
  if (assignee != null) body.assignee = assignee;
  if (due_on != null) body.due_on = due_on;
  if (completed != null) body.completed = completed;

  if (task_id) {
    // Update existing task
    const res = await asanaFetch(
      `/tasks/${task_id}`,
      { method: "PUT", body: JSON.stringify({ data: body }) },
      userId,
    );
    return res.data || res;
  }

  // Create new task
  if (!project_id && !body.projects) {
    throw new Error("project_id is required when creating a new task");
  }
  if (project_id) body.projects = [project_id];
  const res = await asanaFetch(
    "/tasks",
    { method: "POST", body: JSON.stringify({ data: body }) },
    userId,
  );
  return res.data || res;
}

/**
 * Delete a task.
 * @param {{ task_id: string }} input
 * @param {string} [userId]
 */
export async function deleteTask({ task_id } = {}, userId) {
  if (!task_id) throw new Error("task_id is required");
  await asanaFetch(`/tasks/${task_id}`, { method: "DELETE" }, userId);
  return { ok: true, deleted: task_id };
}

// ---------------------------------------------------------------------------
// Sections
// ---------------------------------------------------------------------------

/**
 * Read sections in a project.
 * @param {{ project_id: string }} input
 * @param {string} [userId]
 */
export async function readSections({ project_id } = {}, userId) {
  if (!project_id) throw new Error("project_id is required");
  const res = await asanaFetch(`/projects/${project_id}/sections?opt_fields=name`, {}, userId);
  return { sections: res.data || [] };
}

// ---------------------------------------------------------------------------
// Comments / Stories
// ---------------------------------------------------------------------------

/**
 * Read comments on a task.
 * @param {{ task_id: string, limit?: number }} input
 * @param {string} [userId]
 */
export async function readComments({ task_id, limit = 20 } = {}, userId) {
  if (!task_id) throw new Error("task_id is required");
  const res = await asanaFetch(
    `/tasks/${task_id}/stories?opt_fields=created_by.name,text,created_at,resource_subtype&limit=${limit}`,
    {},
    userId,
  );
  const comments = (res.data || [])
    .filter((s) => s.resource_subtype === "comment_added")
    .map((s) => ({
      author: s.created_by?.name || "unknown",
      text: (s.text || "").slice(0, 1000),
      created_at: s.created_at,
    }));
  return { comments };
}

/**
 * Write a comment on a task.
 * @param {{ task_id: string, text: string }} input
 * @param {string} [userId]
 */
export async function writeComment({ task_id, text } = {}, userId) {
  if (!task_id) throw new Error("task_id is required");
  if (!text) throw new Error("text is required");
  const res = await asanaFetch(
    `/tasks/${task_id}/stories`,
    { method: "POST", body: JSON.stringify({ data: { text } }) },
    userId,
  );
  return res.data || res;
}

/**
 * Edit the text of an existing comment story. Only the original author can edit.
 * @param {{ story_id: string, text: string }} input
 * @param {string} [userId]
 */
export async function editComment({ story_id, text } = {}, userId) {
  if (!story_id) throw new Error("story_id is required");
  if (text == null) throw new Error("text is required");
  const res = await asanaFetch(
    `/stories/${story_id}`,
    { method: "PUT", body: JSON.stringify({ data: { text } }) },
    userId,
  );
  return res.data || res;
}

/**
 * Delete a comment story by its GID. Only the original author can delete.
 * @param {{ story_id: string }} input
 * @param {string} [userId]
 */
export async function deleteComment({ story_id } = {}, userId) {
  if (!story_id) throw new Error("story_id is required");
  await asanaFetch(`/stories/${story_id}`, { method: "DELETE" }, userId);
  return { ok: true, deleted: story_id };
}

// ---------------------------------------------------------------------------
// Credential check
// ---------------------------------------------------------------------------

export async function checkCredentials(userId) {
  try {
    const token = await getToken(userId);
    if (token) {
      return { ok: true, msg: "ASANA_ACCESS_TOKEN is set" };
    }
    return {
      ok: false,
      msg: "Missing ASANA_ACCESS_TOKEN — add it to profile env_vars or process.env.",
    };
  } catch (e) {
    return { ok: false, msg: `Error: ${e.message}` };
  }
}
```
<!-- /GUILDOS-FILE -->
