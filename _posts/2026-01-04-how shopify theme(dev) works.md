---
layout: post
title: "How Shopify theme(and it`s dev env) works"
tags: 
    - Shopify
    - Liquid
    - Javascript
    - CSS
---

{% include mermaidInPost.html %}
<script src="../assets/scripts/mermaid.min.js"></script>

We will walkthrough How Shopify theme and it`s dev environment works. We can understand shopify theme as a kind of site generator, with rich featured editor and contents store. If you came from game industry, you will discover common points between this and game engines like unity and unreal. Lets start with install [Shopify CLI](https://shopify.dev/docs/api/shopify-cli).

Now you can start by making directory and type `shopify theme init`. It will prompt you to login to shopify and make a store, and make a structure for theme.

![result of `shopify theme init`](/assets/images/shopify-theme-init.png)

IMAO, this is too much for beginner. We introduce the smallest theme For the sake of simplicity. It has only 4 files. Paste below files in your theme directory.

/config/settings.json
```json
[
  {
    "name": "Theme settings",
    "settings": [
        {
            "type": "checkbox",
            "id": "dummy_setting",
            "label": "Dummy setting",
            "default": false
        }
    ]
  }
]
```

/layout/theme.liquid
```html
<!doctype html>
<html lang="en">
  <head>
    {{ content_for_header }}
  </head>
  <body>
    {{ content_for_layout }}
  </body>
</html>
```

/sections/main.liquid
```html
<h1>Hello, Shopify Theme 👋</h1>
<p>This is the smallest possible theme.</p>
```

/templates/index.json
```json
{
  "sections": {
    "main": {
      "type": "main"
    }
  },
  "order": ["main"]
}
```

Open your directory in the VSCODE, install Shopify liquid extension, add .gitignore(you can find it out), and start dev by typing `shopify theme dev` at your working directory. You can see preview as below.

![http://localhost:9292](/assets/images/hello-shopify-theme.png)

Let's look int these files. 
`/config/settings.json` : Settings for your theme. At least one settings is required, so we fill in dummy one.
`/layout/theme.liquid` : Actual template. HTML page with template tags waiting to be filled in. `content_for_header` and `content_for_layout` are reserved words.
`/sections/main.liquid` : Looks like DOM component. We can see this piece is placed in the Body of theme.liquid in preview.
`/templates/index.json` : Looks like a kind of metadata.

Before check the online editor, we need to push our theme in out store. Our theme is just a organised directory enough to preview, but Shopify itself doens't know our new theme. `shopify theme push` in your theme directory. It may prompt you to login, authenticate, select store if it is required. Anbd after all, It will prompt to choose select one of existing theme or create new one. I recommend create new one has explicit name.

![pushing theme](/assets/images/push-theme.png)

Then let's go to the online theme editor. We will see how the thing above work actually. Go to `Online store` of your shopify admin page.

![online editor](/assets/images/hello-shopify-theme-online.png)

We see section `Main` is already placed and preview panel right is showing how it woll be rendered. `Add section` button is at down there, but it will show nothing to add. Let's add something to add. Return to the project, and add a file below.

/sections/section.liquid
```html
{% raw %}
<h1>This is a section.</h1>

{% schema %}
{
    "name": "section",
    "tag": "section",
    "class": "section",    
    "presets": [
        {
            "name": "section-instance",
            "blocks": [                        
            ]
        }
    ]
}
{% endschema %}
{% endraw %}
```

We can`t see any of difference on preview yet. Push our theme again, and let's revisit online editor. You may need refresh.

![new dynamic section](/assets/images/new-section.png)

Now we can see `section-instance` in `add-section` popup. Try add, remove(bin button), and add again. Now we know how declare dynamic section, but you may wonder why out first section `main` is not dynamic. Before proceed, save editing theme, with section added, and return to CLI. Type `shopify theme pull` to pull editing theme to local.

/templates/index.json
```json
/*
 * ------------------------------------------------------------
 * IMPORTANT: The contents of this file are auto-generated.
 *
 * This file may be updated by the Shopify admin theme editor
 * or related systems. Please exercise caution as any changes
 * made to this file may be overwritten.
 * ------------------------------------------------------------
 */
{
  "sections": {
    "main": {
      "type": "main",
      "settings": {}
    },
    "section_3CdGKr": {
      "type": "section",
      "name": "section-instance",
      "settings": {}
    }
  },
  "order": [
    "main",
    "section_3CdGKr"
  ]
}

```

Comments saying machine generated is ahead, and section_{hash} object is added. Preview at 9292 should show same page with online preview. We can figure out this file is the state saved in the online editor, and it contains sections and its order. And section file we provided is a entity definition used in online editor. If we remove `presets` object from section schema, we will not be able to add or remove this section from template. This is why `main` section is static here. We omit testing this.

During above steps, We walked through principle workflow of site generation.

- make empty template locally.
- upload template and preview.
- make editable entity.
- upload entity and edit template actually using the entity we made.
- pull back edited template.

This is the way. There are much more options and configurations, but we have to do actually is making components, upload, edit with components, repeat several times, and publish. 
Many of components require dynamic fill-in contents, like product information, and they can be queried only in online editor. So it's hard to accomplish test our components all the way along locally. That's why we have to upload theme and pull back.

Using above steps, let's test out `block`, `snippet`, `asset`. Paste files below. Maybe `snippets` and `assets` directories should be made.

/sections/section.liquid
```html
{% raw %}
{{ 'info-panel.css' | asset_url | stylesheet_tag }}

<h1>This is a section.</h1>

{%- for block in section.blocks -%}
    {%- case block.type -%}
        {%- when 'info-panel' -%}
            {% render 'info-panel', block: block %}
    {%- endcase -%}
{%- endfor -%}    


{% schema %}
{
    "name": "section",
    "tag": "section",
    "class": "section",    
    "blocks": [
        {
            "type": "info-panel",
            "name": "info panel",      
            "settings": [
                {
                    "type": "richtext",
                    "id": "info",
                    "label": "text content"
                }
            ]
        }                                
    ],
    "presets": [
        {
            "name": "section-instance",
            
        }
    ]
}
{% endschema %}
{% endraw %}
```

/snippets/info-panel.liquid
```html
<div class="info-panel">
    {{ block.settings.info }}  	
</div>
```

/assets/info-panel.css
```css
.info-panel {
	background: rgba(200, 200, 200, 1);	
}
```

And push your theme again. Now we can add info-panel under our new section, and modify it's info text.

![text content](/assets/images/text-content.png)

Furthermore We see snippet file is rendered in section, and CSS in assets are loaded with section. If we want to give a kind of behavior, you can place JS in assets and bind them with DOM element rendered. Let's pull back out theme. 

/templates/index.json
```json
/*
 * ------------------------------------------------------------
 * IMPORTANT: The contents of this file are auto-generated.
 *
 * This file may be updated by the Shopify admin theme editor
 * or related systems. Please exercise caution as any changes
 * made to this file may be overwritten.
 * ------------------------------------------------------------
 */
{
  "sections": {
    "main": {
      "type": "main",
      "settings": {}
    },
    "section_3CdGKr": {
      "type": "section",
      "blocks": {
        "info_panel_bUdBqX": {
          "type": "info-panel",
          "settings": {
            "info": "<p>123123123</p>"
          }
        }
      },
      "block_order": [
        "info_panel_bUdBqX"
      ],
      "name": "section-instance",
      "settings": {}
    }
  },
  "order": [
    "main",
    "section_3CdGKr"
  ]
}


```

We see block instance `info_panel_bUdBqX` is placed. Now we can see edited theme in local preview  same as online editor. 

<div class="mermaid">
    ---
        config:
            class:
                hideEmptyMembersBox: true
    ---
    classDiagram    
    theme --> preview : shopify theme dev
    theme --> online : shopify theme push
    theme <-- online : shopify theme pull
    online --> production : publish
    class theme{
        /assets - JS, CSS, or else
        /layout - HTML
        /sections - partial DOM. can have blocks
        /snippets- re-usable DOM
        /templates - my site actually configured using aboves 
    }
    class local_preview{        
        http://localhost:9292
    }
    class online_editor{
        https://admin.shopify.com/store/.../themes/.../editor
    }
    class production{
        https://mydomain.shopify.com
    }
</div>

I`ve wrap up how shopify theme works between online editor and local CLI. Enjoy!!
