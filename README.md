# Writing

This repository holds my public writing from [thomashoneyman.com](https://thomashoneyman.com) so the community can help fix typos, suggest improvements, and update content that has fallen out of date. Feel free to open issues or pull requests if you notice something incorrect or off about any content on the site.

## Using Assets

Images and other data used to generate the content is stored in the `assets` directory. To include an image in the markdown content, link to it with `/images/` as the root:

```md
![image title](/images/2019/image.png)
```

## Templates

I currently use [Hugo](https://gohugo.io) to generate site content. Hugo supports features for templating, including [shortcodes](https://gohugo.io/content-management/shortcodes) in markdown to opt-in to richer content.

### Shortcodes

Currently, my site supports the following shortcodes:

- `{{< external-link "https://example.com" "link text" >}}` is used to include external links in markdown (it adds an icon to indicate this is an external link).
- `{{< table src="data.csv" class="className" >}}` is used to build tables from CSV data located in the `data` directory. It creates headers for the first row in the CSV and table rows for each row afterwards.
- `{{< subscribe >}}` is used to insert an email subscription block

## License

Copyright © 2018-2020 Thomas Honeyman
