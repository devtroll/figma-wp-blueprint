# Theme file templates

Starting points for `/wp-theme`. Replace `{prefix}`, `{text-domain}`, and
`{Theme Name}`. `blueprint/context/coding-standards.md` wins on any conflict.

## style.css

```css
/*
Theme Name: {Theme Name}
Author: {Author}
Description: Custom theme built from Figma.
Version: 1.0.0
Requires at least: 6.5
Requires PHP: 8.1
Text Domain: {text-domain}
*/
```

## functions.php

```php
<?php
defined( 'ABSPATH' ) || exit;

foreach ( array( 'setup', 'helpers', 'enqueue', 'acf', 'blocks' ) as $file ) {
	require_once get_theme_file_path( "inc/{$file}.php" );
}

foreach ( glob( get_theme_file_path( 'inc/post-types/*.php' ) ) ?: array() as $file ) {
	require_once $file;
}
```

## inc/setup.php

```php
<?php
defined( 'ABSPATH' ) || exit;

function {prefix}_setup() {
	add_theme_support( 'title-tag' );
	add_theme_support( 'post-thumbnails' );
	add_theme_support( 'custom-logo' );
	add_theme_support( 'html5', array( 'search-form', 'gallery', 'caption', 'style', 'script', 'navigation-widgets' ) );

	register_nav_menus(
		array(
			'primary' => __( 'Primary Menu', '{text-domain}' ),
			'footer'  => __( 'Footer Menu', '{text-domain}' ),
		)
	);
}
add_action( 'after_setup_theme', '{prefix}_setup' );

// The block editor (Gutenberg) is the standard editor; pages are built from the theme's ACF blocks (inc/blocks.php).
```

## inc/enqueue.php

```php
<?php
defined( 'ABSPATH' ) || exit;

function {prefix}_enqueue_style( $handle, $relative_path, $deps = array() ) {
	$path = get_theme_file_path( $relative_path );
	if ( file_exists( $path ) ) {
		wp_enqueue_style( $handle, get_theme_file_uri( $relative_path ), $deps, (string) filemtime( $path ) );
	}
}

function {prefix}_enqueue_script( $handle, $relative_path ) {
	$path = get_theme_file_path( $relative_path );
	if ( file_exists( $path ) ) {
		wp_enqueue_script(
			$handle,
			get_theme_file_uri( $relative_path ),
			array(),
			(string) filemtime( $path ),
			array(
				'strategy'  => 'defer',
				'in_footer' => true,
			)
		);
	}
}

function {prefix}_enqueue_assets() {
	{prefix}_enqueue_style( '{prefix}-variables', 'assets/css/variables.css' );
	{prefix}_enqueue_style( '{prefix}-base', 'assets/css/base.css', array( '{prefix}-variables' ) );

	foreach ( array( 'components', 'sections' ) as $dir ) {
		foreach ( glob( get_theme_file_path( "assets/css/{$dir}/*.css" ) ) ?: array() as $file ) {
			$name = basename( $file, '.css' );
			{prefix}_enqueue_style( "{prefix}-{$dir}-{$name}", "assets/css/{$dir}/{$name}.css", array( '{prefix}-base' ) );
		}
	}

	{prefix}_enqueue_script( '{prefix}-main', 'assets/js/main.js' );

	foreach ( glob( get_theme_file_path( 'assets/js/sections/*.js' ) ) ?: array() as $file ) {
		$name = basename( $file, '.js' );
		{prefix}_enqueue_script( "{prefix}-section-{$name}", "assets/js/sections/{$name}.js" );
	}
}
add_action( 'wp_enqueue_scripts', '{prefix}_enqueue_assets' );
```

## inc/acf.php

```php
<?php
defined( 'ABSPATH' ) || exit;

function {prefix}_acf_json_save_path() {
	return get_theme_file_path( 'acf-json' );
}
add_filter( 'acf/settings/save_json', '{prefix}_acf_json_save_path' );

function {prefix}_acf_json_load_paths( $paths ) {
	$paths[] = get_theme_file_path( 'acf-json' );
	return $paths;
}
add_filter( 'acf/settings/load_json', '{prefix}_acf_json_load_paths' );

function {prefix}_acf_options_page() {
	if ( function_exists( 'acf_add_options_page' ) ) {
		acf_add_options_page(
			array(
				'page_title' => __( 'Theme Settings', '{text-domain}' ),
				'menu_slug'  => 'theme-settings',
				'capability' => 'edit_theme_options',
				'redirect'   => false,
			)
		);
	}
}
add_action( 'acf/init', '{prefix}_acf_options_page' );

function {prefix}_acf_missing_notice() {
	if ( ! class_exists( 'ACF' ) && current_user_can( 'activate_plugins' ) ) {
		printf(
			'<div class="notice notice-error"><p>%s</p></div>',
			esc_html__( 'This theme requires Advanced Custom Fields PRO.', '{text-domain}' )
		);
	}
}
add_action( 'admin_notices', '{prefix}_acf_missing_notice' );
```

## inc/helpers.php

```php
<?php
defined( 'ABSPATH' ) || exit;

/** Read an ACF option without a fatal error when ACF is inactive. */
function {prefix}_option( $name ) {
	return function_exists( 'get_field' ) ? get_field( $name, 'option' ) : null;
}

/** Inline an SVG from assets/icons/ (trusted theme files only). */
function {prefix}_icon( $name, $class = '' ) {
	$path = get_theme_file_path( "assets/icons/{$name}.svg" );
	if ( ! file_exists( $path ) ) {
		return '';
	}
	$svg = file_get_contents( $path );
	return str_replace( '<svg', '<svg class="' . esc_attr( $class ) . '" aria-hidden="true" focusable="false"', $svg );
}
```

## inc/blocks.php

Starting point. Verify the filters against the installed WordPress and ACF Pro versions.

```php
<?php
defined( 'ABSPATH' ) || exit;

/** Register every ACF block from blocks/<block>/block.json. */
function {prefix}_register_blocks() {
	foreach ( glob( get_theme_file_path( 'blocks/*/block.json' ) ) ?: array() as $file ) {
		register_block_type( dirname( $file ) );
	}
}
add_action( 'init', '{prefix}_register_blocks' );

function {prefix}_block_category( $categories ) {
	array_unshift( $categories, array( 'slug' => '{prefix}-sections', 'title' => __( 'Sections', '{text-domain}' ) ) );
	return $categories;
}
add_filter( 'block_categories_all', '{prefix}_block_category' );

/** Only the theme's own blocks plus the core blocks the plan names. */
function {prefix}_allowed_blocks( $allowed, $context ) {
	$theme = array_map(
		static function ( $file ) {
			return 'acf/' . basename( dirname( $file ) );
		},
		glob( get_theme_file_path( 'blocks/*/block.json' ) ) ?: array()
	);
	$core  = array( 'core/paragraph', 'core/heading', 'core/list', 'core/list-item', 'core/image' );
	return array_merge( $theme, $core );
}
add_filter( 'allowed_block_types_all', '{prefix}_allowed_blocks', 10, 2 );

/**
 * Page block templates: the structure per page template, locked against adding and
 * deleting for editors (moving and hiding stay possible), unlocked for administrators.
 * Map page template slugs to block lists in inc/block-templates.php.
 */
function {prefix}_page_block_template() {
	$screen = get_current_screen();
	if ( ! $screen || 'page' !== $screen->post_type ) {
		return;
	}
	$post_id = isset( $_GET['post'] ) ? absint( $_GET['post'] ) : 0; // phpcs:ignore WordPress.Security.NonceVerification
	$slug    = $post_id ? get_page_template_slug( $post_id ) : '';
	$map     = require get_theme_file_path( 'inc/block-templates.php' ); // slug => array( array( 'acf/hero', array() ), ... )
	$object  = get_post_type_object( 'page' );

	if ( 'free-page.php' === $slug || empty( $map[ $slug ] ) ) {
		return; // free page template (administrators only): no fixed structure.
	}
	$object->template      = $map[ $slug ];
	$object->template_lock = current_user_can( 'manage_options' ) ? false : 'insert';
}
add_action( 'current_screen', '{prefix}_page_block_template' );
```

## page.php

```php
<?php
defined( 'ABSPATH' ) || exit;

get_header();
?>
<main id="main" class="site-main">
	<?php
	while ( have_posts() ) {
		the_post();
		the_content();
	}
	?>
</main>
<?php
get_footer();
```

Blocks print their own markup; there is no section loop. Page templates (Home, About, ...) are
registered with a `Template Name` header and map to a block list in `inc/block-templates.php`.

## header.php (skeleton)

```php
<?php defined( 'ABSPATH' ) || exit; ?>
<!doctype html>
<html <?php language_attributes(); ?>>
<head>
	<meta charset="<?php bloginfo( 'charset' ); ?>">
	<meta name="viewport" content="width=device-width, initial-scale=1">
	<?php wp_head(); ?>
</head>
<body <?php body_class(); ?>>
<?php wp_body_open(); ?>
<a class="skip-link visually-hidden-focusable" href="#main"><?php esc_html_e( 'Skip to content', '{text-domain}' ); ?></a>
<header class="site-header" data-section="site-header">
	<div class="container site-header__inner">
		<!-- logo, nav from wp_nav_menu( array( 'theme_location' => 'primary', 'container' => false ) ), toggle button -->
	</div>
</header>
```

## footer.php (skeleton)

```php
<?php defined( 'ABSPATH' ) || exit; ?>
<footer class="site-footer">
	<div class="container site-footer__inner">
		<!-- content from {prefix}_option(), footer menu -->
	</div>
</footer>
<?php wp_footer(); ?>
</body>
</html>
```

## assets/js/main.js (menu toggle pattern)

```js
{
	const toggle = document.querySelector('[data-menu-toggle]');
	const menu = document.querySelector('[data-menu]');

	if (toggle && menu) {
		const setOpen = (open) => {
			toggle.setAttribute('aria-expanded', String(open));
			menu.classList.toggle('is-open', open);
			document.body.classList.toggle('menu-open', open);
		};

		toggle.addEventListener('click', () => {
			setOpen(toggle.getAttribute('aria-expanded') !== 'true');
		});

		document.addEventListener('keydown', (event) => {
			if (event.key === 'Escape' && toggle.getAttribute('aria-expanded') === 'true') {
				setOpen(false);
				toggle.focus();
			}
		});
	}
}
```
