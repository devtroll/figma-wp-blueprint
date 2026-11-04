# ACF local JSON shapes

ACF fills defaults for omitted field settings when it loads local JSON, so keep
files readable: include `key`, `label`, `name`, `type`, and the settings that
matter. Set `modified` to the current Unix time on every edit.

## ACF block: block.json

One file per block in `blocks/<block>/block.json`; the theme registers it with `register_block_type()`.

```json
{
  "$schema": "https://schemas.wp.org/trunk/block.json",
  "apiVersion": 3,
  "name": "acf/hero",
  "title": "Hero",
  "category": "{prefix}-sections",
  "icon": "cover-image",
  "description": "Page header",
  "acf": { "mode": "preview", "renderTemplate": "template-parts/sections/hero.php" },
  "supports": { "anchor": false, "align": false, "customClassName": false, "html": false, "color": false, "typography": false, "spacing": false },
  "style": "file:../../assets/css/sections/hero.css",
  "script": "file:../../assets/js/sections/hero.js"
}
```

The block key is English snake_case or kebab-case (`hero`), the `title` and
field labels use the site's editing language. Variants of a Figma pattern are a `type` select in the
layout group.

## Block content group: group_block_hero.json

Content fields only: what the client may change (texts, images, lists) plus
`hide_block`. Templates read them with `get_field()` (inside a block render
template ACF resolves the block's own fields).

```json
{
  "key": "group_block_hero",
  "title": "Block: Hero",
  "fields": [
    { "key": "field_block_hero_hide_block", "label": "Hide block", "name": "hide_block", "type": "true_false", "ui": 1 },
    { "key": "field_block_hero_pretitle", "label": "Pretitle", "name": "pretitle", "type": "text" },
    { "key": "field_block_hero_heading", "label": "Heading", "name": "heading", "type": "text", "required": 1 },
    { "key": "field_block_hero_text", "label": "Text", "name": "text", "type": "wysiwyg", "tabs": "visual", "toolbar": "basic", "media_upload": 0 },
    { "key": "field_block_hero_image", "label": "Bild", "name": "image", "type": "image", "return_format": "id", "preview_size": "medium" },
    { "key": "field_block_hero_button", "label": "Button", "name": "button", "type": "link", "return_format": "array" },
    {
      "key": "field_block_hero_items",
      "label": "Items",
      "name": "items",
      "type": "repeater",
      "layout": "block",
      "button_label": "Add item",
      "sub_fields": [
        { "key": "field_block_hero_items_title", "label": "Titel", "name": "title", "type": "text" }
      ]
    }
  ],
  "location": [
    [{ "param": "block", "operator": "==", "value": "acf/hero" }]
  ],
  "active": true,
  "modified": 1760000000
}
```

## Block layout group: group_block_hero_layout.json

Design settings, visible to administrators only (the location rule combines the block with the user role).

```json
{
  "key": "group_block_hero_layout",
  "title": "Block: Hero (Layout)",
  "fields": [
    { "key": "field_block_hero_layout_type", "label": "Variant", "name": "type", "type": "select", "choices": { "image_left": "Image left", "image_right": "Image right", "basic": "Basic" }, "default_value": "image_right" }
  ],
  "location": [
    [
      { "param": "block", "operator": "==", "value": "acf/hero" },
      { "param": "current_user_role", "operator": "==", "value": "administrator" }
    ]
  ],
  "active": true,
  "modified": 1760000000
}
```

## Options page: group_theme_settings.json

```json
{
  "key": "group_theme_settings",
  "title": "Theme Settings",
  "fields": [
    { "key": "field_theme_settings_tab_header", "label": "Header", "name": "", "type": "tab" },
    { "key": "field_theme_settings_header_cta", "label": "Header CTA", "name": "header_cta", "type": "link", "return_format": "array" },
    { "key": "field_theme_settings_tab_footer", "label": "Footer", "name": "", "type": "tab" },
    { "key": "field_theme_settings_footer_text", "label": "Footer text", "name": "footer_text", "type": "textarea", "rows": 3, "new_lines": "br" }
  ],
  "location": [
    [{ "param": "options_page", "operator": "==", "value": "theme-settings" }]
  ],
  "active": true,
  "modified": 1760000000
}
```

## CPT fields: group_project_fields.json

```json
{
  "key": "group_project_fields",
  "title": "Project Details",
  "fields": [
    { "key": "field_project_fields_client", "label": "Client", "name": "client", "type": "text" },
    { "key": "field_project_fields_gallery", "label": "Gallery", "name": "gallery", "type": "gallery", "return_format": "id" }
  ],
  "location": [
    [{ "param": "post_type", "operator": "==", "value": "project" }]
  ],
  "position": "acf_after_title",
  "active": true,
  "modified": 1760000000
}
```

## Reading fields safely

| Field type | Read | Output |
| --- | --- | --- |
| text, textarea | `get_field( 'heading' )` | `esc_html( $heading )` |
| wysiwyg | `get_field( 'text' )` | `wp_kses_post( $text )` |
| image (ID) | `get_field( 'image' )` | `wp_get_attachment_image( $image, 'large', false, array( 'class' => 'x__image' ) )` |
| link (array) | `get_field( 'button' )` | `esc_url( $button['url'] )`, `esc_html( $button['title'] )`, `esc_attr( $button['target'] ?: '_self' )` |
| url | `get_field( 'url' )` | `esc_url( $url )` |
| repeater | `have_rows( 'items' )` / `the_row()` / `get_sub_field()` | per sub-field type |
| option | `{prefix}_option( 'footer_text' )` | per field type |
| CPT field | `get_field( 'client' )` inside the loop | per field type |
