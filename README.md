# comfy.guide
Comfortable computer tutorial webpage.

## How to Contribute

Begin by cloning and entering the repository:
```sh
git clone https://github.com/aledenshi/comfy.guide
cd comfy.guide
```

To preview the website as it's being edited, run the following command in the background:
```sh
hugo server --noHTTPCache -D
```

*The `-D` makes it so any articles marked with `draft: true` will still show up in the preview*

Then navigate to [localhost:1313](http://127.0.0.1:1313) to see the site!

### Creating a New Article

Create a new page under the appropriate section:
```sh
## For server-side software
hugo new content/server/softwarename.md

## For client-side software
hugo new content/client/softwarename.md
```

Then, begin editing that file:
```sh
vim content/client/softwarename.md
```
