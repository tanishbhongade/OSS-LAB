# Reproducing the WordPress Login, Registration, and Dashboard Setup

This guide documents the setup already completed. It covers the Docker
Compose WordPress environment, the child theme, its styling, the custom
authentication handlers, and the three existing shortcodes.

> Scope: this guide includes **Home (the WordPress site homepage),
> Login, Register, and Dashboard pages**. It does **not** add a new Home
> shortcode.

## 1. Project structure

The project directory is:

``` text
local-services/wordpress/
├── docker-compose.yml
└── WordPress files (inside the Docker volume)
    └── wp-content/
        └── themes/
            └── my-personal-theme/
                ├── style.css
                └── functions.php
```

The custom theme is named `My Personal Theme` and is a child theme of
`twentytwentyfive`.

## 2. Start WordPress with Docker Compose

Create `docker-compose.yml` in the project directory:

``` yaml
services:
  wordpress:
    image: wordpress
    restart: always
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    volumes:
      - wordpress:/var/www/html

  db:
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:
```

Start the containers:

``` bash
docker compose up -d
```

Open WordPress in your browser:

``` text
http://localhost:8080
```

Complete the normal WordPress installation if this is a fresh
environment.

## 3. Create the child theme

Enter the WordPress container:

``` bash
docker compose exec wordpress bash
```

Go to the themes directory and create the theme folder:

``` bash
cd /var/www/html/wp-content/themes/
mkdir -p my-personal-theme
cd my-personal-theme
```

Create `style.css` with this theme header and the CSS used for the
authentication and dashboard layouts.

### `style.css`

``` css
/*
Theme Name: My Personal Theme
Template: twentytwentyfive
Version: 1.0
*/

/* Authentication pages */

.auth-container {
    max-width: 420px;
    margin: 70px auto;
    padding: 35px;
    background: #ffffff;
    border: 1px solid #e5e7eb;
    border-radius: 12px;
    box-shadow: 0 8px 30px rgba(0, 0, 0, 0.04);
}

.auth-container h2 {
    font-size: 30px;
    margin-bottom: 8px;
    text-align: center;
}

.auth-container p {
    color: #64748b;
    text-align: center;
}

.auth-container form {
    margin-top: 25px;
}

.auth-container label {
    display: block;
    font-weight: 600;
    margin: 15px 0 7px;
}

.auth-container input:not([type="checkbox"]) {
    width: 100%;
    padding: 13px;
    border: 1px solid #d1d5db;
    border-radius: 7px;
    box-sizing: border-box;
    font-size: 15px;
}

.auth-container .auth-remember {
    display: flex;
    align-items: center;
    gap: 8px;
    font-weight: normal;
}

.auth-container button {
    width: 100%;
    padding: 14px;
    margin-top: 20px;
    border: none;
    border-radius: 7px;
    background: #111827;
    color: white;
    font-size: 16px;
    cursor: pointer;
}

.auth-container button:hover {
    background: #374151;
}

.auth-error {
    color: #dc2626 !important;
}

.auth-success {
    color: #15803d !important;
}

/* Dashboard */

.dashboard-container {
    max-width: 850px;
    margin: 60px auto;
    padding: 30px;
    border: 1px solid #e5e7eb;
    border-radius: 12px;
}

.dashboard-links {
    display: flex;
    gap: 15px;
    flex-wrap: wrap;
    margin-top: 30px;
}

.dashboard-links a {
    padding: 12px 20px;
    background: #111827;
    color: #ffffff;
    border-radius: 7px;
    text-decoration: none;
}
```

## 4. Add the theme functionality

Create `functions.php` in the same theme directory.

### `functions.php`

``` php
<?php

function my_personal_theme_styles() {
    wp_enqueue_style(
        'my-personal-theme-style',
        get_stylesheet_uri(),
        array(),
        '1.0'
    );
}

add_action('wp_enqueue_scripts', 'my_personal_theme_styles');

/* =====================================
   CUSTOM AUTHENTICATION
===================================== */

// Redirect users to our custom login page.
function my_auth_login_url() {
    return home_url('/login/');
}

// Handle custom login form submissions.
function my_handle_login() {

    if (
        !isset($_POST['my_login_nonce']) ||
        !wp_verify_nonce(
            sanitize_text_field(wp_unslash($_POST['my_login_nonce'])),
            'my_custom_login'
        )
    ) {
        wp_die('Invalid security token.');
    }

    $credentials = array(
        'user_login'    => sanitize_text_field(
            wp_unslash($_POST['username'])
        ),
        'user_password' => $_POST['password'] ?? '',
        'remember'      => !empty($_POST['remember']),
    );

    $user = wp_signon($credentials, is_ssl());

    if (is_wp_error($user)) {
        wp_safe_redirect(
            add_query_arg('login', 'failed', my_auth_login_url())
        );
        exit;
    }

    wp_safe_redirect(home_url('/dashboard/'));
    exit;
}

add_action(
    'admin_post_nopriv_my_custom_login',
    'my_handle_login'
);

// Handle registration.
function my_handle_registration() {

    if (
        !isset($_POST['my_register_nonce']) ||
        !wp_verify_nonce(
            sanitize_text_field(wp_unslash($_POST['my_register_nonce'])),
            'my_custom_register'
        )
    ) {
        wp_die('Invalid security token.');
    }

    if (!get_option('users_can_register')) {
        wp_die('Registration is currently disabled.');
    }

    $username = sanitize_user(
        wp_unslash($_POST['username'] ?? '')
    );

    $email = sanitize_email(
        wp_unslash($_POST['email'] ?? '')
    );

    $password = $_POST['password'] ?? '';

    if (
        !validate_username($username) ||
        username_exists($username)
    ) {
        wp_die('Invalid or already registered username.');
    }

    if (!is_email($email) || email_exists($email)) {
        wp_die('Invalid or already registered email.');
    }

    if (strlen($password) < 12) {
        wp_die('Password must contain at least 12 characters.');
    }

    $user_id = wp_insert_user(array(
        'user_login' => $username,
        'user_email' => $email,
        'user_pass'  => $password,
        'role'       => 'subscriber',
    ));

    if (is_wp_error($user_id)) {
        wp_die('Registration failed. Please try again.');
    }

    // Send the new account notification to the user.
    wp_send_new_user_notifications($user_id, 'user');

    wp_safe_redirect(home_url('/login/?registered=1'));
    exit;
}

add_action(
    'admin_post_nopriv_my_custom_register',
    'my_handle_registration'
);

// Redirect logged-out visitors away from dashboard.
function my_protect_dashboard() {

    if (
        is_page('dashboard') &&
        !is_user_logged_in()
    ) {
        wp_safe_redirect(my_auth_login_url());
        exit;
    }
}

add_action('template_redirect', 'my_protect_dashboard');

// Login form shortcode.
function my_login_shortcode() {

    if (is_user_logged_in()) {
        return '<p>You are already logged in.
        <a href="' . esc_url(home_url('/dashboard/')) .
        '">Go to Dashboard</a></p>';
    }

    ob_start();
    ?>

    <div class="auth-container">
        <h2>Welcome Back</h2>
        <p>Sign in to continue.</p>

        <?php if (isset($_GET['login'])) : ?>
            <p class="auth-error">
                Invalid username or password.
            </p>
        <?php endif; ?>

        <?php if (isset($_GET['registered'])) : ?>
            <p class="auth-success">
                Account created! You can now log in.
            </p>
        <?php endif; ?>

        <form method="post"
        action="<?php echo esc_url(admin_url('admin-post.php')); ?>">

            <input type="hidden" name="action"
            value="my_custom_login">

            <?php wp_nonce_field(
                'my_custom_login',
                'my_login_nonce'
            ); ?>

            <label>Username or Email</label>
            <input type="text" name="username"
            autocomplete="username" required>

            <label>Password</label>
            <input type="password" name="password"
            autocomplete="current-password" required>

            <label class="auth-remember">
                <input type="checkbox" name="remember" value="1">
                Remember me
            </label>

            <button type="submit">Sign In</button>

        </form>

        <p>
            <a href="<?php echo esc_url(wp_lostpassword_url()); ?>">
                Forgot password?
            </a>
        </p>

        <p>
            Don't have an account?
            <a href="<?php echo esc_url(home_url('/register/')); ?>">
                Register
            </a>
        </p>
    </div>

    <?php
    return ob_get_clean();
}

add_shortcode('my_login', 'my_login_shortcode');

// Registration form shortcode.
function my_register_shortcode() {

    if (is_user_logged_in()) {
        return '<p>You are already registered and logged in.</p>';
    }

    if (!get_option('users_can_register')) {
        return '<p>Registration is currently unavailable.</p>';
    }

    ob_start();
    ?>

    <div class="auth-container">

        <h2>Create Account</h2>
        <p>Join our website.</p>

        <form method="post"
        action="<?php echo esc_url(admin_url('admin-post.php')); ?>">

            <input type="hidden" name="action"
            value="my_custom_register">

            <?php wp_nonce_field(
                'my_custom_register',
                'my_register_nonce'
            ); ?>

            <label>Username</label>
            <input type="text" name="username"
            autocomplete="username" required>

            <label>Email</label>
            <input type="email" name="email"
            autocomplete="email" required>

            <label>Password (minimum 12 characters)</label>
            <input type="password" name="password"
            autocomplete="new-password"
            minlength="12" required>

            <button type="submit">Create Account</button>

        </form>

        <p>
            Already have an account?
            <a href="<?php echo esc_url(home_url('/login/')); ?>">
                Sign In
            </a>
        </p>

    </div>

    <?php
    return ob_get_clean();
}

add_shortcode('my_register', 'my_register_shortcode');

// Dashboard shortcode.
function my_dashboard_shortcode() {

    if (!is_user_logged_in()) {
        return '<p>Please log in to view your dashboard.</p>';
    }

    $user = wp_get_current_user();

    ob_start();
    ?>

    <div class="dashboard-container">

        <h2>
            Welcome, <?php echo esc_html($user->display_name); ?>!
        </h2>

        <p>You are successfully logged in.</p>

        <div class="dashboard-links">

            <a href="<?php echo esc_url(get_edit_profile_url($user->ID)); ?>">
                Edit Profile
            </a>

            <a href="<?php echo esc_url(wp_logout_url(home_url('/'))); ?>">
                Logout
            </a>

        </div>

    </div>

    <?php
    return ob_get_clean();
}

add_shortcode('my_private_dashboard', 'my_dashboard_shortcode');
```

## 5. Activate the theme

In WordPress Admin:

1.  Open **Appearance → Themes**.
2.  Find **My Personal Theme**.
3.  Click **Activate**.

The theme directory should contain:

``` text
my-personal-theme/
├── style.css
└── functions.php
```

## 6. Enable user registration

In WordPress Admin:

1.  Open **Settings → General**.
2.  Find **Membership**.
3.  Enable **Anyone can register**.
4.  Set **New User Default Role** to **Subscriber**.
5.  Save Changes.

Registration is checked in the PHP handler and shortcode, so the
registration form will not be available while registration is disabled.

## 7. Create the three shortcode pages

Open **Pages → Add New** and create the following pages. Add the
corresponding shortcode in each page's content.

  Page title   Page slug     Page content
  ------------ ------------- --------------------------
  Login        `login`       `[my_login]`
  Register     `register`    `[my_register]`
  Dashboard    `dashboard`   `[my_private_dashboard]`

Publish all three pages.

The resulting URLs should be:

``` text
http://localhost:8080/login/
http://localhost:8080/register/
http://localhost:8080/dashboard/
```

The site's Home page is the normal WordPress homepage. No additional
Home shortcode is part of this setup.

## 8. How the authentication flow works

1.  A visitor opens the Register page.
2.  The registration shortcode displays the account creation form if
    WordPress registration is enabled.
3.  The form submits to `admin-post.php` using the `my_custom_register`
    action.
4.  The registration handler checks the nonce, validates the username
    and email, checks the 12-character minimum password length, and
    creates a Subscriber account.
5.  After successful registration, the visitor is redirected to
    `/login/?registered=1`.
6.  The Login page displays the success message and the login form.
7.  The login form submits using the `my_custom_login` action.
8.  On successful login, the visitor is redirected to `/dashboard/`.
9.  Logged-out visitors who open the Dashboard page are redirected to
    `/login/`.
10. The Dashboard shortcode displays the logged-in user's display name,
    an Edit Profile link, and a Logout link.

## 9. Shortcode reference

These are the three shortcodes implemented in `functions.php`:

  Shortcode                  Purpose
  -------------------------- -----------------------------------------
  `[my_login]`               Displays the custom login form
  `[my_register]`            Displays the custom registration form
  `[my_private_dashboard]`   Displays the logged-in user's dashboard

## 10. Test the setup

Use a private/incognito browser window or log out before testing.

-   Open `/register/` and create a test account.
-   Confirm that registration redirects to `/login/?registered=1`.
-   Log in using the new account.
-   Confirm that successful login redirects to `/dashboard/`.
-   Confirm that the dashboard shows the user's display name.
-   Test the Edit Profile link.
-   Test Logout and confirm that it returns to the homepage.
-   While logged out, open `/dashboard/` and confirm that it redirects
    to `/login/`.
-   While logged out, open `/login/` and `/register/` and confirm that
    their forms display.

## 11. Important implementation notes

-   The login and registration forms use WordPress nonces.
-   Registration checks whether WordPress user registration is enabled.
-   New users are assigned the `subscriber` role.
-   Registration enforces a minimum password length of 12 characters.
-   The dashboard page has a logged-out redirect, and the dashboard
    shortcode also checks login status.
-   The login form label says "Username or Email", but the current
    handler passes the submitted value directly as `user_login` to
    `wp_signon()`. The supplied implementation does not separately
    convert an email address to a username.
-   The Docker Compose file uses example database credentials. For a
    production deployment, replace them with secure credentials and
    review the WordPress and database configuration before exposing the
    site publicly.

