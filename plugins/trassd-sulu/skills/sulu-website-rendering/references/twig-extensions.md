# Sulu Twig functions & filters

Sulu provides these Twig functions and filters in addition to the standard Twig
set, for use in website templates. Prefer them over hand-rolled content/media
lookups.

## CoreBundle

### Functions
- `sulu_article_load`
- `sulu_content_path`
- `sulu_content_root_path`
- `sulu_page_breadcrumb`
- `sulu_page_load`
- `sulu_page_navigation_root_tree`
- `sulu_page_navigation_flat`
- `sulu_page_navigation_root_flat`
- `sulu_page_navigation_tree`

### Filters
- `sulu_util_multisort`

## SnippetBundle
- `sulu_snippet_load_by_area`

## MediaBundle
- `sulu_get_media_url`
- `sulu_resolve_media`
- `sulu_resolve_medias`

## TagBundle
- `sulu_tags`
- `sulu_tag_url`
- `sulu_tag_url_append`
- `sulu_tag_url_clear`

## CategoryBundle
- `sulu_categories`
- `sulu_category_url`
- `sulu_category_url_append`
- `sulu_category_url_remove`
- `sulu_category_url_toggle`
- `sulu_category_url_clear`

## ContactBundle
- `sulu_resolve_contact`

## SecurityBundle
- `sulu_resolve_user`
