# Joomla User Registration, Login and Protected Dashboard

## Complete practical guide for reproducing the website in an examination

This guide documents the complete workflow for creating a simple Joomla
website with:

-   A Home page
-   User registration
-   User login and logout
-   A dashboard visible only to registered users
-   A My Profile page
-   A separate Dashboard Menu displayed inside the dashboard
-   A blue-and-white design using the Cassiopeia template and CSS

The procedure uses the Joomla Administrator interface and CSS. It does
**not** require custom PHP or JavaScript.

> **Note:** Joomla menu labels can vary slightly between Joomla
> versions. Look for the equivalent menu item if your version uses a
> different label.

------------------------------------------------------------------------

## 1. Start Joomla

1.  Start Apache and MySQL in XAMPP (if using a local XAMPP
    installation).
2.  Open your Joomla website in a browser, for example:
    `http://localhost:8080/`
3.  Open the Administrator login page:
    `http://localhost:8080/administrator`
4.  Sign in using your administrator account.
5.  Confirm that the **Cassiopeia** site template is selected.

If Joomla is not installed yet, first create a database in phpMyAdmin,
open the Joomla installer, enter the database connection details, create
the Super User account, and complete installation.

------------------------------------------------------------------------

## 2. Enable user registration

1.  In Administrator, go to **Users → Manage**.
2.  Open **Options**.
3.  Find **Allow User Registration**.
4.  Set it to **Yes**.
5.  Save and close.

This allows visitors to create their own Joomla accounts.

------------------------------------------------------------------------

## 3. Create the main menu items

Open **Menus → Main Menu**. Create or check the following menu items.

### 3.1 Home

1.  Select **New**.
2.  For Menu Item Type, choose **Articles → Featured Articles** or the
    appropriate Home page item for your site.
3.  Set the title to `Home`.
4.  Set Access to `Public`.
5.  Save and close.

### 3.2 Login

1.  Select **New**.
2.  Choose **Users → Login Form** as the Menu Item Type.
3.  Set the title to `Login`.
4.  Set Access to `Public`.
5.  Save and close.

### 3.3 Register

1.  Select **New**.
2.  Choose **Users → Registration Form**.
3.  Set the title to `Register`.
4.  Set Access to `Public`.
5.  Save and close.

### 3.4 User Dashboard

Create an article first, as described in Section 5. Then:

1.  Go to **Menus → Main Menu → New**.
2.  Set the title to `User Dashboard`.
3.  Choose **Articles → Single Article**.
4.  Select the `User Dashboard` article.
5.  Set **Access** to `Registered`.
6.  Save and close.

The Registered access level is important: users must log in before they
can view this page.

### 3.5 My Profile

1.  Go to **Menus → Main Menu → New**.
2.  Choose **Users → Edit User Profile**.
3.  Set the title to `My Profile`.
4.  Set Access to `Registered`.
5.  Set Status to `Published`.
6.  Save and close.

This opens the profile-editing page for the currently logged-in user.

### 3.6 Logout

1.  Go to **Menus → Main Menu → New**.
2.  Choose **Users → Logout**.
3.  Set the title to `Logout`.
4.  Set Access to `Registered`.
5.  Set Status to `Published`.
6.  Save and close.

This creates a Joomla logout link.

> If you want My Profile and Logout only inside the dashboard, do not
> add them to the public Main Menu. Create them in the separate
> Dashboard Menu in Section 6. If they are already in the Main Menu,
> they may also appear in the top navigation, as they did during this
> project.

------------------------------------------------------------------------

## 4. Create the Home page

1.  Go to **Content → Articles → New**.
2.  Set the title to `Home`.
3.  Add a welcome heading and a short introduction.
4.  Add feature descriptions if required.
5.  Save and close.
6.  Make sure the Home menu item points to the correct Home article or
    featured-articles view.

Example introductory content:

``` html
<section class="home-hero">
    <h1>Welcome to Our Website</h1>
    <p>Create an account or log in to access your personal dashboard.</p>
</section>
```

You can add more content, such as feature cards, below the introduction.

------------------------------------------------------------------------

## 5. Create the protected User Dashboard article

1.  Go to **Content → Articles → New**.
2.  Set the title to `User Dashboard`.
3.  Set the category to an appropriate category, such as
    `Uncategorised`.
4.  Add the following content in the editor's **Code/Source** view:

``` html
<div class="dashboard-welcome">
    <h2>Welcome to Your Dashboard!</h2>
    <p>You have successfully logged in. Manage your account from here.</p>
</div>

{loadposition dashboard-menu}
```

5.  Set the article's **Access** to `Registered`.
6.  Set Status to `Published`.
7.  Save and close.

The `{loadposition dashboard-menu}` instruction is a Joomla content
plugin tag. It tells Joomla to display modules assigned to the module
position named `dashboard-menu`.

**Important:** Enter the loadposition instruction as normal article
content. Do not put it inside a Markdown code fence or HTML `<code>`
element.

### Link the dashboard article to the Main Menu

If you have not already created the menu item:

1.  Go to **Menus → Main Menu → New**.
2.  Set the title to `User Dashboard`.
3.  Choose **Articles → Single Article**.
4.  Select the `User Dashboard` article.
5.  Set Access to `Registered`.
6.  Save and close.

Both the article and its menu item should be restricted to Registered
users.

------------------------------------------------------------------------

## 6. Create a separate Dashboard Menu

A separate menu keeps dashboard links organised and allows the menu
module to display only the dashboard options.

1.  Go to **Menus → Manage → Add New Menu**.
2.  Enter:
    -   **Title:** `Dashboard Menu`
    -   **Menu Type:** `dashboard-menu`
3.  Save and close.

### 6.1 Add My Profile to Dashboard Menu

1.  Go to **Menus → Dashboard Menu → New**.
2.  Choose **Users → Edit User Profile**.
3.  Set the title to `My Profile`.
4.  Set Access to `Registered`.
5.  Set Status to `Published`.
6.  Save and close.

### 6.2 Add Logout to Dashboard Menu

1.  Go to **Menus → Dashboard Menu → New**.
2.  Choose **Users → Logout**.
3.  Set the title to `Logout`.
4.  Set Access to `Registered`.
5.  Set Status to `Published`.
6.  Save and close.

The Dashboard Menu should now contain two published items: My Profile
and Logout.

------------------------------------------------------------------------

## 7. Create and configure the Dashboard Navigation module

This is the key step that makes the Dashboard Menu appear inside the
dashboard article.

1.  Go to **Content → Site Modules** (in some Joomla versions, use
    **System → Manage → Site Modules**).
2.  Select **New**.
3.  Choose the **Menu** module.
4.  Set its title to `Dashboard Navigation`.

### 7.1 Module tab

Set the following values:

  Setting       Value
  ------------- ----------------------------------
  Select Menu   `Dashboard Menu`
  Base Item     Dashboard Menu root, if required
  Start Level   `1`
  End Level     `All`
  Status        `Published`
  Position      `dashboard-menu`
  Access        `Registered`
  Show Title    Optional; usually No

**Very important:** The module's **Position** field must be
`dashboard-menu`. In this project, the Position field initially showed
`None`; that prevented the module from being loaded by the article tag.

If `dashboard-menu` is not listed in the Position dropdown, try entering
the custom position name in the field and pressing Enter. The exact way
to enter a custom position depends on the Joomla version and
administrator interface.

### 7.2 Menu Assignment tab

1.  Choose **Only on the pages selected**.
2.  Select the `User Dashboard` menu item.
3.  Save the module.

This limits the module to the dashboard page. If the module does not
appear during testing, temporarily choose **On all pages** to test
whether the module itself works. The loadposition tag is still only
present in the dashboard article.

### 7.3 Advanced tab

Set:

  Setting           Value
  ----------------- ------------------
  Target Position   `dashboard-menu`
  Module Class      `dashboard-menu`

Keep the two concepts separate:

-   **Position** (Module tab): where Joomla places the module.
-   **Target Position** (Advanced tab): a module setting that can be
    used for module rendering/position targeting.
-   **Module Class** (Advanced tab): a CSS class used to style the
    module.

For the article's `{loadposition dashboard-menu}` tag to find the
module, the module's **Position** field must match `dashboard-menu`.

Save and close.

------------------------------------------------------------------------

## 8. Enable the Load Modules content plugin

The loadposition tag depends on Joomla's Load Modules content plugin.

1.  Go to **System → Manage → Plugins**.
2.  Search for `Content - Load Modules`.
3.  Open the plugin.
4.  Set Status to **Enabled**.
5.  Save and close.

If this plugin is disabled, Joomla may display the
`{loadposition dashboard-menu}` text instead of rendering the module, or
the module may not appear.

------------------------------------------------------------------------

## 9. Add CSS styling to Cassiopeia

Use the Cassiopeia template's custom CSS file so the styling remains
separate from Joomla's core files.

1.  Go to **System → Site Templates → Cassiopeia Details and Files**
    (the label can vary by Joomla version).
2.  Open or create `css/user.css`.
3.  Add the CSS below.
4.  Save the file.

### 9.1 Welcome banner and heading

``` css
.dashboard-welcome {
    background: linear-gradient(120deg, #184b82, #2878cc);
    color: #ffffff;
    padding: 45px;
    border-radius: 18px;
    margin: 25px 0;
}

.dashboard-welcome h2 {
    color: #ffffff !important;
    font-size: 38px;
    font-weight: bold;
    margin-bottom: 15px;
}

.dashboard-welcome p {
    color: #ffffff;
    font-size: 18px;
}
```

### 9.2 Dashboard menu cards

``` css
.dashboard-menu ul {
    list-style: none;
    display: flex;
    flex-wrap: wrap;
    gap: 20px;
    padding: 0;
    margin: 20px 0;
}

.dashboard-menu li {
    flex: 1 1 220px;
    background: #ffffff;
    border-top: 5px solid #2878cc;
    border-radius: 15px;
    padding: 25px;
    box-shadow: 0 5px 20px rgba(0, 0, 0, 0.08);
}

.dashboard-menu a {
    color: #184b82 !important;
    font-size: 18px;
    font-weight: bold;
    text-decoration: none;
    display: block;
}

.dashboard-menu a:hover {
    color: #2878cc !important;
}
```

### 9.3 Optional: hide article information on the dashboard

The dashboard screenshot showed article information such as author,
category, publication date and hits. To hide this information, you can
turn off the relevant display options in the article or menu item
settings:

-   Open the User Dashboard menu item or article options.
-   Find the **Show Author**, **Show Category**, **Show Publish Date**,
    and **Show Hits** settings.
-   Set them to **Hide** where available.
-   Save and refresh.

The exact controls depend on the Joomla version and whether the options
are inherited from global article settings.

------------------------------------------------------------------------

## 10. Test the website

Test the complete user journey, not just the appearance.

### Test A: Public visitor

1.  Open the website in a private/incognito browser window or log out.
2.  Confirm that Home, Login and Register are accessible.
3.  Try opening the User Dashboard URL.
4.  Joomla should require login or deny access because the dashboard is
    restricted to Registered users.

### Test B: Registration

1.  Open the Register page.
2.  Fill in the required account details.
3.  Submit the registration form.
4.  Complete any activation step if the site requires it.

### Test C: Login

1.  Open the Login page.
2.  Enter the newly registered account credentials.
3.  Submit the form.
4.  Confirm that the user is logged in.

### Test D: Dashboard

1.  Open User Dashboard.
2.  Confirm that the welcome banner appears.
3.  Confirm that the Dashboard Menu appears below it.
4.  Click My Profile and confirm that the current user's profile page
    opens.
5.  Return to the dashboard.
6.  Click Logout and confirm that the user is logged out.

After CSS or module changes, refresh the browser with **Ctrl + F5** to
bypass cached styles.

------------------------------------------------------------------------

## 11. Troubleshooting checklist

  -----------------------------------------------------------------------------------------
  Problem                             What to check
  ----------------------------------- -----------------------------------------------------
  Registration page is unavailable    Users → Manage → Options → Allow User Registration =
                                      Yes

  Dashboard is visible to guests      Dashboard article and dashboard menu item Access =
                                      Registered

  `{loadposition dashboard-menu}`     Enable Content - Load Modules plugin
  appears as plain text               

  Dashboard menu does not appear      Module is Published; its Position is exactly
                                      `dashboard-menu`; plugin is enabled

  Menu module displays the wrong      Select Menu must be `Dashboard Menu`, not `Main Menu`
  links                               

  Module is not displayed on the      Check Menu Assignment and select User Dashboard
  dashboard                           

  Dashboard menu links are invisible  Check the CSS class and link colour in `user.css`

  Welcome heading is dark on a blue   Add
  background                          `.dashboard-welcome h2 { color: #fff !important; }`

  Changes do not appear               Save all changes and press Ctrl + F5
  -----------------------------------------------------------------------------------------

### The most important configuration to remember

For this project, the Dashboard Navigation module must have:

-   **Select Menu:** `Dashboard Menu`
-   **Position:** `dashboard-menu`
-   **Access:** `Registered`
-   **Status:** `Published`
-   **Module Class:** `dashboard-menu`

And the User Dashboard article must contain:

``` text
{loadposition dashboard-menu}
```

The **module Position** and the **loadposition tag** must use exactly
the same name.

------------------------------------------------------------------------

## 12. Quick examination revision

Remember the sequence:

1.  Install/start Joomla and open Administrator.
2.  Enable user registration.
3.  Create Home, Login and Register menu items.
4.  Create the User Dashboard article.
5.  Restrict the dashboard article to Registered users.
6.  Create the User Dashboard menu item and set Access to Registered.
7.  Create a separate Dashboard Menu.
8.  Add My Profile and Logout to Dashboard Menu.
9.  Create a Menu module called Dashboard Navigation.
10. Select Dashboard Menu in the module.
11. Set the module Position to `dashboard-menu`.
12. Set Access to Registered and publish the module.
13. Assign the module to the User Dashboard page.
14. Enable Content - Load Modules.
15. Insert `{loadposition dashboard-menu}` in the dashboard article.
16. Add CSS to Cassiopeia's `user.css`.
17. Test registration, login, dashboard access, profile editing and
    logout.

**Result:** A Joomla website with user registration, login, a protected
user dashboard, profile management and logout, styled with the
Cassiopeia template.
