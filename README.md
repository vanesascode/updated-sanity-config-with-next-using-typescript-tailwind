# UPDATED SANITY CONFIG with NEXT using TYPESCRIPT (& TAILWIND)

Many concepts from how Sanity.io worked a few months ago, are explained in this [repo](https://github.com/vanesascode/configuring-sanity-next-tailwind-typescript)

## 🔷installing Next.js

`npx create-next-app@latest nextjs-app --typescript --tailwind --eslint --app --no-src-dir --import-alias="@/*"`

`cd nextjs-app`

## 🔷installing Sanity.io

`cd nextjs-app`

`npx sanity@latest init --env --create-project "Next.js Live Preview" --dataset production`

> Would you like to add configuration files for a Sanity project in this Next.js folder?
> Yes

> Do you want to use TypeScript?
> Yes

> Would you like an embedded Sanity Studio?
> Yes

> Would you like to use the Next.js app directory for routes?
> Yes

> What route do you want to use for the Studio?
> /studio

> Select project template to use
> Blog (schema)

> Would you like to add the project ID and dataset to your .env file?
> Yes

## 🔷env folder

Rename it into `.env.local`

## 🔷Sanity Client

Creating the new Sanity project inside our Next.js application created a Sanity Client inside /nextjs-app/sanity/lib/`client.ts`

```
import { createClient } from 'next-sanity'

import { apiVersion, dataset, projectId, useCdn } from '../env'

export const client = createClient({
  apiVersion,
  dataset,
  projectId,
  useCdn,
})

```

👉 The `useCdn` variable is used as a configuration option when creating a client object with the createClient function. It determines whether the client should use the `Content Delivery Network (CDN)` to fetch data from the Sanity API.

If useCdn is set to true, the client will use the CDN, which provides cached responses and can improve performance. If it is set to false, the client will bypass the CDN and fetch data directly from the Sanity API, which can be useful during development or when working with draft content.

In the provided code snippet, the value of useCdn is imported from the env file, so the specific value depends on the content of that file.

--- let's continue:

In spite of this great `client.ts` file, to fully benefit from advanced features like draft mode, revalidation and caching we'll need to create a new file that integrates all these features:

## 🔷Sanity Fetch

🔹 Install these additional packages:

`npm install suspend-react server-only`

The `suspend-react` package provides a way to suspend rendering components in React. It allows you to temporarily pause rendering and resume it later, which can be useful for optimizing performance in certain scenarios.

The `server-only` flag indicates that you only want to install the package for use on the server-side. This can be useful if you have specific server-side rendering requirements and don't need the package to be included in client-side code.

🔹 Create a new file for the sanityFetch function (`SanityFetch.ts` file)

(See and copy the contents of the `SanityFetch.ts` file in this repository)

## 🔷GROQ queries

Create a new file inside the lib folder: `queries.ts`:

```
import { groq } from "next-sanity";

// Get all posts
export const postsQuery = groq`*[_type == "post" && defined(slug.current)]{
    _id, title, slug
  }`;

// Get a single post by its slug
export const postQuery = groq`*[_type == "post" && slug.current == $slug][0]{
    title, mainImage, body
  }`;

// Get all post slugs
export const postPathsQuery = groq`*[_type == "post" && defined(slug.current)][]{
    "params": { "slug": slug.current }
  }`;

```

These queries are written in the `GROQ (Graph-Relational Object Queries)` language, which is a query language specifically designed for querying structured documents in Sanity.

## 🔷Rendering the data and passing it as props

So now that we have all we need, we import it in the main `page.tsx` file inside the app folder:

```
import { SanityDocument } from "next-sanity";
import Posts from "@/app/_components/Posts";
import { postsQuery } from "@/sanity/lib/queries";
import { sanityFetch } from "@/sanity/lib/sanityFetch";

export default async function Home() {
  const posts = await sanityFetch<SanityDocument[]>({ query: postsQuery });

  return <Posts posts={posts} />;
}
```

- The posts variable is declared and assigned the result of the sanityFetch function call.

- The sanityFetch function is called with an object parameter containing the query property set to postsQuery, which is a GROQ query for retrieving posts.

- The sanityFetch function is an asynchronous function that fetches data from the Sanity API. In this case, it is fetching an array of SanityDocument objects representing posts.

- The generic type <SanityDocument[]> is used to specify the expected type of the fetched data.

- The next line of code returns the Posts component and passes the posts variable as a `prop`. The posts prop is used within the Posts component to render the list of posts.

This is the Posts component:

```
import Link from "next/link";
import type { SanityDocument } from "@sanity/client";

export default function Posts({ posts = [] }: { posts: SanityDocument[] }) {
  const title = posts.length === 1 ? `1 Post` : `${posts.length} Posts`;

  return (
    <main className="container mx-auto grid grid-cols-1 divide-y divide-blue-100">
      <h1 className="text-2xl p-4 font-bold">{title}</h1>
      {posts.map((post) => (
        <Link
          key={post._id}
          href={post.slug.current}
          className="p-4 hover:bg-blue-50"
        >
          <h2>{post.title}</h2>
        </Link>
      ))}
    </main>
  );
}

```

See how we have been importing the `type SanityDocument` from the next-sanity package. So we don't have to define it ourselves 👏

- The Posts component is declared as a function component that takes an object parameter with a `destructured posts prop`. The posts prop is set to an empty array by default in case it is not provided.

- For each post object, a `Link component` is rendered. The Link component has a key attribute set to the post.\_id value and an `href attribute` set to post.slug.current.

- `post.slug.current` : Post refers to an individual post object in the posts array. Slug is a property of the post object. Current is a property of the slug property. So, post.slug.current is a string value representing the current slug of the post.

A `slug` is typically a URL-friendly version of a title or identifier used to uniquely identify a resource. It is commonly used in websites to create human-readable and SEO-friendly URLs.

## 🔷Portable Text

Portable Text is a field type provided by the Sanity CMS (Content Management System) that allows you to `store rich text content in a structured and portable format`. The @portabletext/react package provides React components and utilities for rendering and working with portable text fields in a React application.

You must install it:

- [x] Run: `npm i @portabletext/react@latest`

Once you have installed the package, you can import the necessary components and utilities from @portabletext/react in your code and use them to render and work with portable text fields in your React application.

## Display individual posts

Create a file called `Post.tsx` in your \_components folder:

```
"use client";

import Image from "next/image";
import imageUrlBuilder from "@sanity/image-url";
import { SanityDocument } from "@sanity/client";
import { PortableText } from "@portabletext/react";
import { client } from "@/sanity/lib/client";

const builder = imageUrlBuilder(client);

export default function Post({ post }: { post: SanityDocument }) {
  return (
    <main className="container mx-auto prose prose-lg p-4">
      <h1>{post.title}</h1>
      {post?.mainImage ? (
        <Image
          className="float-left m-0 w-1/3 mr-4 rounded-lg"
          src={builder.image(post.mainImage).width(300).height(300).url()}
          width={300}
          height={300}
          alt={post?.mainImage?.alt}
        />
      ) : null}
      {post?.body ? <PortableText value={post.body} /> : null}
    </main>
  );
}
```

- The `builder` is used to generate URLs for images stored in Sanity.
- The component receives an object parameter with a `destructured post prop` of type SanityDocument.

Now we create a a new route to query for the post by its slug:

- Inside the app folder, we create a folder called [slug], and inside we create a file called `page.tsx`

---

In Next.js, when a folder is written between square brackets ([]), it is used to denote a dynamic route or a catch-all route.

`Dynamic Routes`: When a folder is named with square brackets, it indicates that the route is dynamic and can match multiple values. For example, if you have a folder named [id] inside the pages directory, it means that any path segment that matches this pattern will be captured as a dynamic parameter named id. You can then access this dynamic parameter in your page component through the useRouter hook provided by Next.js. Example:

```
pages/
  [id]/
    index.js
```

`Catch-All Routes`: If the folder name is written as [[...slug]], it represents a catch-all route. This means that it can match any number of path segments and capture them as a dynamic parameter named slug. Example:

``pages/
  [[...slug]]/
    index.js`

```
If a user visits the URL /category/posts/123, the index.js file inside the [[...slug]] folder will be responsible for rendering the page. The slug parameter will be an array ['category', 'posts', '123'].
```

---

Let's continue. In the `page.tsx` file we add:

```
import { SanityDocument } from "@sanity/client";
import Post from "@/app/_components/Post";
import { postPathsQuery, postQuery } from "@/sanity/lib/queries";
import { sanityFetch } from "@/sanity/lib/sanityFetch";
import { client } from "@/sanity/lib/client";

// Prepare Next.js to know which routes already exist
export async function generateStaticParams() {
  // Important, use the plain Sanity Client here
  const posts = await client.fetch(postPathsQuery);

  return posts;
}

export default async function Page({ params }: { params: any }) {
  const post = await sanityFetch<SanityDocument>({ query: postQuery, params });

  return <Post post={post} />;
}
```

👉 This code represents a Next.js page component that fetches a single post from Sanity CMS based on the params prop and renders it using the Post component. The generateStaticParams function is used to prepare Next.js to generate static routes based on the available post paths. So:

- The `generateStaticParams` function is asynchronously defined. It is used to prepare Next.js to know which routes already exist. It fetches the post paths using the client from @/sanity/lib/client and returns them.

- The export default statement defines the Page function component. It receives an object parameter with a `destructured params prop` of type any.

- Inside the component, the `sanityFetch function` is used to fetch a single post from Sanity CMS. The sanityFetch function takes an object parameter with query and params properties. The query property is set to postQuery, which likely represents a query to fetch a single post from Sanity. The params property is set to the params prop received by the component.

- The fetched post is then rendered by passing it as a `prop` to the Post component.

## Displaying images with next/image

Update next.config.ts so that next/image will load images from the Sanity CDN:

```
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'cdn.sanity.io',
      },
    ],
  },
  // ...other config settings
};

module.exports = nextConfig;
```

In here we are defining a `nextConfig object` that configures the Next.js image optimization and loading behavior.

- The pattern object has two properties: protocol and hostname. The protocol property is set to 'https', indicating that the images should be fetched using the `HTTPS protocol`. The hostname property is set to `cdn.sanity.io`, specifying the hostname (part of a URL that identifies the server where a resource is hosted) where the remote images are hosted.

## Displaying rich text with @tailwindcss/typography

The Portable Text field in the Studio is being rendered into HTML by the <PortableText /> component.

Install the Tailwind CSS Typography package to quickly apply beautiful default styling:

`npm install -D @tailwindcss/typography`

Update your tailwind.config.js file's plugins to include it:

```
module.exports = {
  // ...other settings
  plugins: [require('@tailwindcss/typography')],
}
```

This package styles the prose class names in the <Post /> component.
