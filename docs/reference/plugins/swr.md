---
description: Plugin for generating SWR query and mutation hooks
sidebar_position: 5
---

# @zenstackhq/swr

:::info
If you're looking for generating hooks for [Tanstack Query](https://tanstack.com/query/latest), please checkout the [`@zenstackhq/tanstack-query`](/docs/reference/plugins/tanstack-query) plugin.
:::

The `@zenstackhq/swr` plugin generates [SWR](https://swr.vercel.app/) hooks that call into the CRUD services provided by the [server adapters](/docs/category/server-adapters).

The hooks syntactically mirror the APIs of a standard Prisma client, including the function names and shapes of parameters (hooks directly use types generated by Prisma).

To use the generated hooks, you need to install "swr" version 2.0.0 or above.

## Context Provider

The plugin generates a React context provider which you can use to configure the behavior of the hooks. The following options are available on the provider:

- endpoint

    The endpoint to use for the queries. Defaults to "/api/model".

- fetch

    A custom `fetch` function to use for the queries. Defaults to the browser's built-in `fetch`. 

Example for using the context provider:

```tsx
import { FetchFn, Provider as ZenStackHooksProvider } from '../lib/hooks';

// custom fetch function that adds a custom header
const myFetch: FetchFn = (url, options) => {
    options = options ?? {};
    options.headers = {
        ...options.headers,
        'x-my-custom-header': 'hello world',
    };
    return fetch(url, options);
};

function MyApp({ Component, pageProps: { session, ...pageProps } }: AppProps) {
    return (
        <ZenStackHooksProvider value={{ endpoint: '/api/model', fetch: myFetch }}>
            <AppContent />
        </ZenStackHooksProvider>
    );
}

export default MyApp;

```

## Options

| Name    | Type   | Description                                             | Required | Default |
| ------- | ------ | ------------------------------------------------------- | -------- | ------- |
| output  | String | Output directory (relative to the path of ZModel)                                        | Yes      |         |
| useSuperJson  | Boolean | Use [superjson](https://github.com/blitz-js/superjson) for data serialization                                        | No      | false        |

## Example

Here's a quick example with a blogging app. You can find a fully functional Todo app example [here](https://github.com/zenstackhq/sample-todo-nextjs).

### Schema

```zmodel title='/schema.zmodel'
plugin hooks {
  provider = '@zenstackhq/swr'
  output = "./src/lib/hooks"
}

model User {
  id            String    @id @default(cuid())
  email         String
  posts         Post[]

  // everyone can signup, and user profile is also publicly readable
  @@allow('create,read', true)
}

model Post {
  id        String @id @default(cuid())
  title     String
  published Boolean @default(false)
  author    User @relation(fields: [authorId], references: [id])
  authorId  String

  // author has full access
  @@allow('all', auth() == author)

  // logged-in users can view published posts
  @@allow('read', auth() != null && published)
}
```

### Using Query and Mutation Hooks

```tsx title='/src/components/posts.tsx'
import type { Post } from '@prisma/client';
import { useFindManyPost, useMutatePost } from '../lib/hooks';

// post list component
const Posts = ({ userId }: { userId: string }) => {
    const { createPost } = useMutatePost();

    // list all posts that're visible to the current user, together with their authors
    const { data: posts } = useFindManyPost({
        include: { author: true },
        orderBy: { createdAt: 'desc' },
    });

    async function onCreatePost() {
        await createPost({
            data: {
                title: 'My awesome post',
                authorId: userId,
            },
        });
    }

    return (
        <>
            <button onClick={onCreatePost}>Create</button>
            <ul>
                {posts?.map((post) => (
                    <li key={post.id}>
                        {post.title} by {post.author.email}
                    </li>
                ))}
            </ul>
        </>
    );
};
```

### Using Infinite Query

See [SWR's documentation](https://swr.vercel.app/docs/pagination) for more details.

```tsx title='/src/components/posts.tsx'
import type { Post } from '@prisma/client';
import { useInfiniteFindManyPost } from '../lib/hooks';

// post list component with infinite loading
const Posts = ({ userId }: { userId: string }) => {

    const PAGE_SIZE = 10;

    const { data: pages, size, setSize } = useInfiniteFindManyPost(
        (pageIndex, previousPageData) => {
            if (previousPageData && !previousPageData.length) {
                return null;
            }
            return {
                include: { author: true },
                orderBy: { createdAt: 'desc' },
                take: PAGE_SIZE,
                skip: pageIndex * PAGE_SIZE,
            };
        }
    );

    const isEmpty = pages?.[0]?.length === 0;
    const isReachingEnd = isEmpty || (pages && pages[pages.length - 1].length < PAGE_SIZE);

    return (
        <>
            <ul>
                {pages?.map((posts, index) => (
                    <React.Fragment key={index}>
                        {posts?.map((post) => (
                            <li key={post.id}>
                                {post.title} by {post.author.email}
                            </li>
                        ))}
                    </React.Fragment>
                ))}
            </ul>

            {!isReachingEnd && (
                <button onClick={() => setSize(size + 1)}>
                    Load more
                </button>
            )}
        </>
    );
};
```
