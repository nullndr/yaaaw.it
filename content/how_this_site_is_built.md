---
title: How this site is built
date: 2024-09-29
description: A technical explanation of how this site works
author: nullndr
---

# How this site is built

This site is built with [remix.run](https://remix.run). There is no database for the posts, instead the posts are written directly in [MDX](https://mdxjs.com/). 

The transformation from MDX to the component is done with the following function:

```typescript
type FrontMatter = {
  title: string;
  description: string;
  published: Date;
  isFeatured: boolean;
};

export const getMdxFile = async (file: string) => {
  const filePath = path.join(process.cwd(), `posts/${file}.mdx`);
  const postContent = (await readfile(filepath)).tostring() 
  return bundleMDX<FrontMatter>({
    source: postContent,
    mdxOptions(options) {
      return {
        rehypePlugins: [...(options.rehypePlugins ?? [])],
        remarkPlugins: [
          ...(options.remarkPlugins ?? []),
          [
            remarkCodeHike,
            {
              theme: "one-dark-pro",
              lineNumbers: true,
              showCopyButton: true,
              autoImport: true,
            },
          ],
        ],
      };
    },
  });
};
```

The function simply reads the content of the post, delagating the real transformation to [mdx-bundler](https://github.com/kentcdodds/mdx-bundler).

The deploy is done with [Coolify](https://github.com/coollabsio/coolify), running on an hetzener vps.