---
title: "Building a Headless CMS with Astro and GitHub API"
date: "2026-07-14"
tags: ["astro", "github", "typescript"]
pinned: true
---

Integrating a headless content strategy doesn't always require expensive SaaS solutions. For developers, a GitHub repository can act as an incredibly effective, version-controlled database. 

In this article, we'll dive into how to fetch, parse, and render Markdown files directly from your GitHub repository using **Astro**, **TypeScript**, and the **GitHub REST API**.

### The Architecture

Instead of compiling your Markdown files at build-time statically inside your Astro project, you can fetch them dynamically or on-demand from a remote GitHub repository. This decouples your content from your codebase.

Here's how the pipeline works:
1. Fetch the git tree of your repository recursively using the GitHub API.
2. Filter for files in your specified posts path (`posts/`).
3. Batch-fetch the raw content of each Markdown file.
4. Parse frontmatter and compile Markdown to HTML securely.

### The Code Implementation

Below is a robust TypeScript implementation that implements a full-featured remote GitHub post loader:

```typescript
import matter from 'gray-matter';
import { marked } from 'marked';
import sanitizeHtml from 'sanitize-html';

export type Post = {
  id: string;
  slug: string;
  title: string;
  date: string;
  tags: string[];
  pinned: boolean;
  body: string;
  html: string;
};

let postsPromise: Promise<Post[]> | undefined;

type GitTreeItem = { path: string; type: string };
type GitTreeResponse = { tree?: GitTreeItem[]; truncated?: boolean };

const requiredVariables = ['GITHUB_OWNER', 'GITHUB_REPO', 'GITHUB_BRANCH'] as const;

function env(name: string): string | undefined {
  const values: Record<string, string | undefined> = {
    GITHUB_OWNER: import.meta.env.GITHUB_OWNER,
    GITHUB_REPO: import.meta.env.GITHUB_REPO,
    GITHUB_BRANCH: import.meta.env.GITHUB_BRANCH,
    GITHUB_POSTS_PATH: import.meta.env.GITHUB_POSTS_PATH,
    GITHUB_TOKEN: import.meta.env.GITHUB_TOKEN,
  };
  return values[name];
}

function getConfig() {
  const missing = requiredVariables.filter((name) => !env(name));
  if (missing.length > 0) {
    throw new Error(`Missing GitHub configuration: ${missing.join(', ')}`);
  }

  return {
    owner: env('GITHUB_OWNER') as string,
    repo: env('GITHUB_REPO') as string,
    branch: env('GITHUB_BRANCH') as string,
    postsPath: (env('GITHUB_POSTS_PATH') || 'posts').replace(/^\/+|\/+$/g, ''),
    token: env('GITHUB_TOKEN'),
  };
}

async function githubFetch<T>(url: string): Promise<T> {
  const token = env('GITHUB_TOKEN');
  const response = await fetch(url, {
    headers: {
      Accept: 'application/vnd.github+json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      'X-GitHub-Api-Version': '2022-11-28',
    },
  });

  if (!response.ok) {
    throw new Error(`GitHub request failed (${response.status}): ${url}`);
  }
  return response.json() as Promise<T>;
}

function asSlug(path: string, postsPath: string): string {
  return path
    .replace(new RegExp(`^${postsPath}/`), '')
    .replace(/\.md$/i, '')
    .replace(/\//g, '-')
    .toLowerCase()
    .replace(/[^a-z0-9-]+/g, '-')
    .replace(/^-|-$/g, '');
}

function validateFrontmatter(data: Record<string, unknown>, path: string): string {
  const date = data.date instanceof Date ? data.date.toISOString().slice(0, 10) : String(data.date || '');
  if (typeof data.title !== 'string' || !/^\d{4}-\d{2}-\d{2}$/.test(date)) {
    throw new Error(`Invalid title or date frontmatter in ${path}. Expected title and date: YYYY-MM-DD.`);
  }
  if (!Array.isArray(data.tags) || data.tags.some((tag) => typeof tag !== 'string')) {
    throw new Error(`Invalid tags frontmatter in ${path}. Expected an array of strings.`);
  }
  if (typeof data.pinned !== 'boolean') {
    throw new Error(`Invalid pinned frontmatter in ${path}. Expected true or false.`);
  }
  return date;
}

export async function getPosts(): Promise<Post[]> {
  if (postsPromise) return postsPromise;
  postsPromise = loadPosts();
  return postsPromise;
}

async function loadPosts(): Promise<Post[]> {
  const config = getConfig();
  const treeUrl = `https://api.github.com/repos/${config.owner}/${config.repo}/git/trees/${encodeURIComponent(config.branch)}?recursive=1`;
  const tree = await githubFetch<GitTreeResponse>(treeUrl);
  if (tree.truncated) throw new Error('GitHub returned a truncated file tree; reduce the repository size or posts path.');

  const files = (tree.tree || []).filter(
    (item) => item.type === 'blob' && item.path.startsWith(`${config.postsPath}/`) && item.path.toLowerCase().endsWith('.md'),
  );
  if (files.length === 0) return [];

  const posts = await Promise.all(files.map(async (file) => {
    const rawUrl = `https://raw.githubusercontent.com/${config.owner}/${config.repo}/${encodeURIComponent(config.branch)}/${file.path.split('/').map(encodeURIComponent).join('/')}`;
    const source = await fetch(rawUrl, {
      headers: config.token ? { Authorization: `Bearer ${config.token}` } : undefined,
    });
    if (!source.ok) throw new Error(`Unable to read Markdown file ${file.path} (${source.status}).`);
    const parsed = matter(await source.text());
    const date = validateFrontmatter(parsed.data, file.path);
    const slug = asSlug(file.path, config.postsPath);
    return {
      id: file.path,
      slug,
      title: parsed.data.title as string,
      date,
      tags: parsed.data.tags as string[],
      pinned: parsed.data.pinned as boolean,
      body: parsed.content,
      html: sanitizeHtml(await marked.parse(parsed.content), {
        allowedTags: sanitizeHtml.defaults.allowedTags.concat(['img']),
        allowedAttributes: { ...sanitizeHtml.defaults.allowedAttributes, img: ['src', 'alt', 'title'] },
      }),
    } satisfies Post;
  }));

  const slugs = new Set<string>();
  for (const post of posts) {
    if (slugs.has(post.slug)) throw new Error(`Duplicate post slug detected: ${post.slug}`);
    slugs.add(post.slug);
  }
  return posts.sort((a, b) => b.date.localeCompare(a.date));
}
```

### Highlights of this setup:
- **Zero Local Footprint:** Your blog content resides cleanly in your content GitHub repo.
- **Strict Validation:** Frontmatter types are strictly validated runtime so you never break your site layout on a bad publish.
- **Bulletproof Caching:** Uses `postsPromise` so you only hit the GitHub API once during server-side renders or build pipelines.
- **Sanitized HTML:** Employs `sanitize-html` allowing standard tags and images but blocking malicious scripts.