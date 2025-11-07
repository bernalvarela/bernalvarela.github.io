# Bernal Varela's Personal Blog

This repository contains the source code for my personal blog, which is accessible at **[https://bernalvarela.is-a.dev](https://bernalvarela.is-a.dev)**.

The blog is built using the static site generator [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.

## How to Add a New Post

To add a new post to the blog, follow these steps:

1.  **Create a new Markdown file** inside the `content/posts/` directory. The filename will be part of the post's URL (e.g., `my-new-post.md`).

2.  **Add the front matter** to the top of the file. This is a YAML block that contains metadata about the post. Here is a basic example:

    ```yaml
    ---
    title: "Your Post Title"
    date: 2024-10-27T10:00:00+02:00
    draft: false
    tags: ["Technology", "Programming", "Java"]
    description: "A brief summary of your post."
    ---
    ```

    - `title`: The title of the post.
    - `date`: The publication date and time.
    - `draft`: Set to `false` to make the post public.
    - `tags`: A list of relevant tags.
    - `description`: A short summary for SEO and previews.

3.  **Write the content** of your post in Markdown format below the front matter.

4.  **Commit and push the changes** to the `main` branch of this repository. GitHub Actions is configured to automatically build and deploy the new version of the site.
