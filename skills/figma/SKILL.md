# figma

<!-- GUILDOS-SKILL name="figma" -->

<!-- GUILDOS-DESC -->
Read design pages, frames, and assets from Figma via the figma weapon (REST API). Discover pages, read a named page's full node tree, list every top-level frame/component, and export frames/components as image URLs by node id. Primary file: Boster-Design-Master-File (NMfOvoGgMVFPYM4nLtN8zD). Requires FIGMA_ACCESS_TOKEN.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/figma/SKILL.md -->
```markdown
---
name: figma
description: >-
  Read design pages, frames, and assets from Figma via the figma weapon (REST
  API). Discover pages, read a named page's full node tree, list every
  top-level frame/component, and export frames/components as image URLs by node
  id. Primary file: Boster-Design-Master-File (NMfOvoGgMVFPYM4nLtN8zD). Requires
  FIGMA_ACCESS_TOKEN.
---

# Figma

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/figma/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto figma <action>` (or read the source directly).

## Actions

- **exportFrames** — Export frames/components as image URLs by node id.
- **readComponents** — List every top-level frame/component across all pages.
- **readPage** — Get the full node tree for one named page.
- **readPages** — List the pages in the master Figma file (or a given fileKey).
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/figma/index.js -->
```js
/**
 * Figma skill book — pure text-prompt registry. Read design pages, frames, and
 * assets from Figma. All logic lives in the `figma` weapon
 * (`libs/weapon/figma/index.js`); this book only says which weapon function to
 * call and when. No JS logic here.
 */

export const skillBook = {
  id: "figma",
  title: "Figma",
  description:
    "Read design pages, frames, and assets from Figma via the figma weapon (REST API). " +
    "Primary file: Boster-Design-Master-File (NMfOvoGgMVFPYM4nLtN8zD). Requires FIGMA_ACCESS_TOKEN.",
  toc: {
    readPages: "List the pages in the master Figma file (or a given fileKey).",
    readPage: "Get the full node tree for one named page.",
    exportFrames: "Export frames/components as image URLs by node id.",
    readComponents: "List every top-level frame/component across all pages.",
  },

  readPages: `
Call \`readPages\` from the figma weapon (\`@/libs/weapon/figma\`): \`readPages(userId, { fileKey? })\`.
Defaults to the Boster Design Master File (\`NMfOvoGgMVFPYM4nLtN8zD\`) when no fileKey is given.
Returns \`{ pages: [{ id, name, childCount }], fileKey }\`. Use this first to discover what pages exist
before a targeted \`readPage\` or \`exportFrames\`.
`.trim(),

  readPage: `
Call \`readPage(userId, { pageName, fileKey?, depth? })\` from the figma weapon. Resolves the named page,
then fetches its full node tree (default depth 3). Returns \`{ page, fileKey }\`, or an error listing the
available page names if \`pageName\` doesn't match. Read layout structure here before exporting frames.
`.trim(),

  exportFrames: `
Call \`exportFrames(userId, { nodeIds, fileKey?, format?, scale? })\` from the figma weapon.
\`nodeIds\` is an array of Figma node IDs (e.g. \`'1234:5678'\`); format defaults to \`png\`, scale to \`2\`.
Returns \`{ images: { nodeId: url }, fileKey }\`. Get node IDs from \`readPage\`/\`readComponents\` first.
`.trim(),

  readComponents: `
Call \`readComponents(userId, { fileKey? })\` from the figma weapon. Returns
\`{ components: [{ id, name, page, type }], fileKey }\` — every top-level frame/component across all pages.
Use it to discover what's available before a targeted \`readPage\` or \`exportFrames\`.
`.trim(),
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/figma/index.js -->
```js
/**
 * Figma weapon — Figma REST API connector for reading files, exporting assets,
 * and listing projects.
 *
 * Auth: FIGMA_ACCESS_TOKEN from profiles.env_vars or process.env.
 */

export const toc = {
  readFile: "Read a Figma file's document tree.",
  readNodes: "Read specific nodes from a Figma file by id.",
  readExport: "Export images of Figma nodes (PNG/SVG/etc).",
  searchProjects: "Search Figma projects within a team.",
  searchFiles: "Search Figma files within a project.",
  readComments: "Read comments on a Figma file.",
  readPages: "List all pages in a file (defaults to the Boster master file) — id, name, child count.",
  readPage: "Get the full node tree for a single named page.",
  exportFrames: "Export frames/components as image URLs by node id.",
  readComponents: "List all top-level frames/components across every page of a file.",
};
import { database } from "@/libs/council/database";
import { skillActionOk, skillActionErr } from "../../skill_book/actionResult.js";

const API_BASE = "https://api.figma.com/v1";

/**
 * Primary design file — Boster Design Master File. Default for the agent-facing
 * read* helpers below when no fileKey is given.
 * URL: https://www.figma.com/design/NMfOvoGgMVFPYM4nLtN8zD/Boster-Design-Master-File
 */
const PRIMARY_FILE_KEY = "NMfOvoGgMVFPYM4nLtN8zD";

// ---------------------------------------------------------------------------
// Auth
// ---------------------------------------------------------------------------

async function getToken(userId) {
  if (process.env.FIGMA_ACCESS_TOKEN) return process.env.FIGMA_ACCESS_TOKEN;
  if (!userId) return null;
  const db = await database.init("service");
  const { data } = await db
    .from("profiles")
    .select("env_vars")
    .eq("id", userId)
    .single();
  return data?.env_vars?.FIGMA_ACCESS_TOKEN ?? null;
}

async function figmaFetch(path, opts = {}, userId) {
  const token = await getToken(userId);
  if (!token) throw new Error("Missing FIGMA_ACCESS_TOKEN");
  const url = path.startsWith("http") ? path : `${API_BASE}${path}`;
  const res = await fetch(url, {
    ...opts,
    headers: {
      "X-Figma-Token": token,
      "Content-Type": "application/json",
      ...opts.headers,
    },
  });
  if (!res.ok) {
    const body = await res.text();
    throw new Error(`Figma API ${res.status}: ${body.slice(0, 500)}`);
  }
  return res.json();
}

// ---------------------------------------------------------------------------
// Files
// ---------------------------------------------------------------------------

/**
 * Read a Figma file's metadata and structure.
 * @param {{ fileKey: string, depth?: number }} input
 * @param {string} [userId]
 */
export async function readFile({ fileKey, depth } = {}, userId) {
  if (!fileKey) throw new Error("fileKey is required");
  const params = new URLSearchParams();
  if (depth != null) params.set("depth", String(depth));
  return figmaFetch(`/files/${fileKey}?${params}`, {}, userId);
}

/**
 * Read specific nodes from a Figma file.
 * @param {{ fileKey: string, nodeIds: string[] }} input
 * @param {string} [userId]
 */
export async function readNodes({ fileKey, nodeIds } = {}, userId) {
  if (!fileKey) throw new Error("fileKey is required");
  if (!Array.isArray(nodeIds) || nodeIds.length === 0) throw new Error("nodeIds is required");
  const params = new URLSearchParams({ ids: nodeIds.join(",") });
  return figmaFetch(`/files/${fileKey}/nodes?${params}`, {}, userId);
}

// ---------------------------------------------------------------------------
// Export
// ---------------------------------------------------------------------------

/**
 * Export renderings of nodes as images.
 * @param {{ fileKey: string, nodeIds: string[], format?: string, scale?: number }} input
 * @param {string} [userId]
 * @returns {Promise<{ images: Record<string, string> }>} — node ID to image URL map
 */
export async function readExport({ fileKey, nodeIds, format = "png", scale = 2 } = {}, userId) {
  if (!fileKey) throw new Error("fileKey is required");
  if (!Array.isArray(nodeIds) || nodeIds.length === 0) throw new Error("nodeIds is required");
  const params = new URLSearchParams({
    ids: nodeIds.join(","),
    format,
    scale: String(scale),
  });
  return figmaFetch(`/images/${fileKey}?${params}`, {}, userId);
}

// ---------------------------------------------------------------------------
// Projects & Team
// ---------------------------------------------------------------------------

/**
 * List projects in a team.
 * @param {{ teamId: string }} input
 * @param {string} [userId]
 */
export async function searchProjects({ teamId } = {}, userId) {
  if (!teamId) throw new Error("teamId is required");
  return figmaFetch(`/teams/${teamId}/projects`, {}, userId);
}

/**
 * List files in a project.
 * @param {{ projectId: string }} input
 * @param {string} [userId]
 */
export async function searchFiles({ projectId } = {}, userId) {
  if (!projectId) throw new Error("projectId is required");
  return figmaFetch(`/projects/${projectId}/files`, {}, userId);
}

// ---------------------------------------------------------------------------
// Comments
// ---------------------------------------------------------------------------

/**
 * Read comments on a file.
 * @param {{ fileKey: string }} input
 * @param {string} [userId]
 */
export async function readComments({ fileKey } = {}, userId) {
  if (!fileKey) throw new Error("fileKey is required");
  return figmaFetch(`/files/${fileKey}/comments`, {}, userId);
}

// ---------------------------------------------------------------------------
// Credential check
// ---------------------------------------------------------------------------

export async function checkCredentials(userId) {
  try {
    const token = await getToken(userId);
    if (token) {
      return { ok: true, msg: "FIGMA_ACCESS_TOKEN is set" };
    }
    return {
      ok: false,
      msg: "Missing FIGMA_ACCESS_TOKEN — generate a personal access token at figma.com/developers and add it to profile env_vars.",
    };
  } catch (e) {
    return { ok: false, msg: `Error: ${e.message}` };
  }
}

// ---------------------------------------------------------------------------
// Agent-facing read helpers (skillActionOk/Err envelope). Default fileKey is the
// Boster Design Master File. Driven by the `figma` skill book's howTos.
// ---------------------------------------------------------------------------

export async function readPages(a, b) {
  const [userId, input] = typeof a === "string" ? [a, b ?? {}] : [undefined, a ?? {}];
  const fileKey = input?.fileKey || PRIMARY_FILE_KEY;
  try {
    const data = await readFile({ fileKey, depth: 1 }, userId);
    const pages = (data?.document?.children || []).map((p) => ({
      id: p.id,
      name: p.name,
      childCount: p.children?.length ?? 0,
    }));
    return skillActionOk({ pages, fileKey });
  } catch (e) {
    return skillActionErr(e.message);
  }
}

export async function readPage(a, b) {
  const [userId, input] = typeof a === "string" ? [a, b ?? {}] : [undefined, a ?? {}];
  if (!input?.pageName) return skillActionErr("pageName required");
  const fileKey = input?.fileKey || PRIMARY_FILE_KEY;
  try {
    const meta = await readFile({ fileKey, depth: 1 }, userId);
    const pages = meta?.document?.children || [];
    const page = pages.find((p) => p.name.toLowerCase() === input.pageName.toLowerCase());
    if (!page) {
      const names = pages.map((p) => p.name).join(", ");
      return skillActionErr(`Page "${input.pageName}" not found. Available: ${names}`);
    }
    const nodes = await readNodes({ fileKey, nodeIds: [page.id] }, userId);
    const pageNode = nodes?.nodes?.[page.id]?.document ?? page;
    return skillActionOk({ page: pageNode, fileKey });
  } catch (e) {
    return skillActionErr(e.message);
  }
}

export async function exportFrames(a, b) {
  const [userId, input] = typeof a === "string" ? [a, b ?? {}] : [undefined, a ?? {}];
  if (!Array.isArray(input?.nodeIds) || input.nodeIds.length === 0)
    return skillActionErr("nodeIds (array) required");
  const fileKey = input?.fileKey || PRIMARY_FILE_KEY;
  try {
    const data = await readExport(
      { fileKey, nodeIds: input.nodeIds, format: input.format ?? "png", scale: input.scale ?? 2 },
      userId
    );
    return skillActionOk({ images: data?.images ?? {}, fileKey });
  } catch (e) {
    return skillActionErr(e.message);
  }
}

export async function readComponents(a, b) {
  const [userId, input] = typeof a === "string" ? [a, b ?? {}] : [undefined, a ?? {}];
  const fileKey = input?.fileKey || PRIMARY_FILE_KEY;
  try {
    const data = await readFile({ fileKey, depth: 2 }, userId);
    const components = [];
    for (const page of data?.document?.children || []) {
      for (const node of page.children || []) {
        components.push({ id: node.id, name: node.name, page: page.name, type: node.type });
      }
    }
    return skillActionOk({ components, fileKey });
  } catch (e) {
    return skillActionErr(e.message);
  }
}
```
<!-- /GUILDOS-FILE -->
