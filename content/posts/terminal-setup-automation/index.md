---
title: 'Terminal Setup Automation with GNU Stow'
date: 2024-09-17T20:15:57-07:00
author: "Darren Victoriano"
hidemeta: false
comments: false
disableHLJS: false # to disable highlightjs
disableShare: true
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowWordCount: false
ShowRssButtonInSectionTermList: true
UseHugoToc: true
# canonicalURL: "https://canonical.url/to/page"
# aliases: ["/first"]
# weight: 1
draft: false
tags: ["MacOS", "Setup", "Terminal", "Zsh", "Stow", "Automation"]
cover:
    image: "<image path/url>" # should be 640 x 330
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    # hiddenList: true  # only hide list view
    # hidden: false # only hide on current single page
---
If you checked out my post on [Terminal Setup]({{< ref "/posts/terminal-setup" >}} "Terminal Setup"), this is a follow-up to that. I’ve gone through several iterations of setting it up on both my personal and work laptops, and honestly, it’s not as fun the second time around. Since automation is my thing, in this tutorial, we’ll automate the whole terminal setup process using the following tools:
* GNU Stow
* MAS
* MacOS defaults and PlistBuddy

If you want to know more about these awesome tools, then read on!

## GNU Stow
[GNU Stow](https://www.gnu.org/software/stow/) is a symlink manager. A symlink (short for symbolic link) is a type of file in Unix-like OS that serves as a reference to another file (similar to a shortcut file in Windows). GNU Stow will help manage our dotfiles/config files (like .zshrc and .p10k.zsh) so that we can store the actual config files in a different location and create a symlink in our home directory (since the system expects dotfiles to exist there). This allows us to compile all our config files and version control it using git and upload it on github as a bragging right.

The idea is to create a folder in our home directory (I’ll name mine `.dotfiles`) where we’ll store all our config files. Then, we’ll invoke the stow command to create the symlinks for us. The stow command expects a specific file directory format, as shown below:
```bash
~/dotfiles/
├── zsh/
│   └── .zshrc
├── p10k/
│   └── .p10k.zsh
```

Some dev tools have more complex config locations, such as `vim` or `neovim`, which store their config files inside `~/.config/`. In these cases, structure the dotfiles folder like this:
```bash
~/dotfiles/
├── zsh/
│   └── .zshrc
├── git/
│   └── .gitconfig
├── p10k/
│   └── .p10k.zsh
├── vim/
│   ├── .vimrc
│   └── .vim/
│       ├── plugins/
│       └── colors/
└── nvim/
    └── .config/
        └── nvim/
            └── init.vim
```

## GNU Stow: Step-by-Step
1. Install GNU Stow via homebrew `brew install stow`
2. Create your directories `mkdir -p ~/.dofiles/zsh ~/.dotfiles/p10k`
3. Navigate to you `home` directory `cd ~`
4. Move your config files to the new directory:
    ```bash
	mv .zshrc ~/.dotfiles/zsh
	mv .p10k.zsh ~/.dotfiles/p10k
    ```
5. Change directory to the `.dotfiles` folder: `cd ~/.dotfiles`
6. Invoke the stow command with the folder name:
	```bash
	stow zsh
	stow p10k
	```
7.  You should now see a symlink files in you home directory.
>Note: You can update your config either from the symlink file in `~/` or in `~/.dotfiles`, personally I prefer updating it in the `~/.dofiles` direcrtory so that I can git push it immediately.

## MAS: Apple Appstore from the command line
If you're like me and not all the GUI applications you use are available through Homebrew Cask, fortunately, there's [mas-cli](https://github.com/mas-cli/mas), a simple command-line interface for the Mac App Store. It's designed for scripting and automation. The GitHub repo has excellent documentation on how to use it. But if you are pressed for time, we'll only going to be using these two commands:
* `mas list` this will show all installed application with their productID
* `mas install <product_id>` this will install the app using the productID

## MacOS `defaults` and `PListBuddy`
Now, in my previous tutorial in [Aesthetic Enhancements]({{< ref "/posts/terminal-setup#aesthetic-enhancements" >}} "Aesthetic Enhancements") section, we had to manually change some settings from the iTerm2 application. Since MacOS stores all user preferences of each application in a `.plist` file saved here:
```
~/Library/Preferences/<domain_name_of_the_app>
```
We can automate updating this configuration in the command line using `defaults` and `PListBuddy` built-in tools.
* `defaults` is a higher-level tool primarily used to interact with the macOS system’s user defaults database, which stores user preferences. It provides a simpler interface to read from and write to plist files but is mainly designed for handling user settings.
* `PlistBuddy` is a tool specifically designed to work directly with plist files. It allows us to perform various operations like adding, deleting, and modifying plist entries with fine-grained control. It's more flexible when you need to edit deeply nested structures or complex data types.

First we use `defaults` to find out what the domain name of the app we want to update the user preferences of, I will be using `iTerm2` for this example. We can do this using the `defaults domains` which will list all domains that stores user preferences, each application typically has its own domain, then we'll use `grep` in conjunctions to filter the result like this:
```bash
defaults domains | grep iterm2
```
This command will print out all the domains and highlight what we `grep` like this:
![iterm2-grep-result](images/iterm2-grep-result.webp)

Now that we know what `iTerm2`'s domain, we can read its content using this command:
```
defaults read com.googlecode.iterm2
```
This will print out all the user preferences for `iTerm2`, this example below is a simplified version, there are a lot of lines that has been redacted.
```bash
{
    HapticFeedbackForEsc = 0;
    HideScrollbar = 0;
    HotkeyMigratedFromSingleToMulti = 1;
    NSNavLastRootDirectory = "~/Downloads";
    NSNavPanelExpandedSizeForOpenMode = "{800, 448}";
    NSOverlayScrollersFallBackForAccessoryViews = 0;
    NSQuotedKeystrokeBinding = "";
    NSRepeatCountBinding = "";
    NSScrollAnimationEnabled = 0;
    "New Bookmarks" = (
        {
            // reacted lines of code
            "Character Encoding" = 4;
            "Close Sessions On End" = 1;
            Columns = 100;
            "Draw Powerline Glyphs" = 0;
            Rows = 30;
        }
    )

}
```
For configurations in the root like `HapticFeedbackForEsc` or `HideScrollbar`, we use `defaults write` to update the value like this:
```
defaults write com.googlecode.iterm2 "HideScrollbar" 1
```
as for nested value like `Character Encoding` or `Columns`, unfortunately `defaults` aren't capable of updating plist value so we will use `PListBuddy` for this. But `PListBuddy` is not in our path so to invoke it, we need to provide the full path which is located in:
```bash
/usr/libexec/PlistBuddy
```
Here's the syntax for updating a value
```bash
/usr/libexec/PlistBuddy -c "Set :<Key_Name> '<New_Value>'" <domain_location>

```
Here's an example of updating the `Columns`
```bash
/usr/libexec/PlistBuddy -c "Set :'New Bookmarks':0:'Columns' 200" ~/Library/Preferences/com.googlecode.iterm2
```

During my automation test run, I ran my installation script(you'll see this in the following section) on a freshly reset Mac and some of the user properties aren't initialized in the plist file yet so I had to add them. You can add using this syntax:
```bash
/usr/libexec/PlistBuddy -c "Add :'<Key_Name>' <data_type> <value>" <domain_location>
```
Example:
```bash
/usr/libexec/PlistBuddy -c "Add :'New Bookmarks':0:'Columns' integer 200" ~/Library/Preferences/com.googlecode.iterm2
```

### How do I know what plist property to change?
Good question, I struggled to answer this as well and after trial and error the easiest method I found it to do a simple `diff`.
Here's how:
1. First save the current plist configuration to a text file:
    ```
    defaults read com.googlecode.iterm2 > original.txt
    ```
2. Then, open the app, in our case iTerm2, then manually change 1 settings from the user preferences.
3. After that, save the new plist to another file
    ```
    defaults read com.googlecode.iterm2 > updated.txt
    ```
4. then now you can just compare 2 files and see what changed:
    ```
    diff original.txt updated.txt
    ```
5. Rinse and repeat.

## The Fun Part
Automation! Before proceeding, upload your `/.dotfiles` to GitHub as we'll be utilizing github for our installation script. [Here’s mine](https://github.com/DarrenVictoriano/dotfiles). I will go over the automation script in the next section.

### Assumptions:
* Homebrew is installed
* Xcode Command Line Tools are installed.
* Git is installed (should come with Xcode Command Line Tools).
>Disclaimer: Importing and Enabling of Catpuccin theme in iTerm2 is still a manual process 🥺

### Automation Script Overview:
We will use a bash script and break it down into multiple files for readability.

1. `_scripts/brew.sh` - Installs all the brew formulas and casks that I need.
2. `_scripts/appstore.sh` - Installs apps from the Apple AppStore using `mas-cli`.
3. `_scripts/ohmyzsh.sh` - Installs oh-my-zsh and all the plugins (nothing special here).
4. `_scripts/app_settings.sh` - Update iTerm2 and other app’s settings using `defaults` and `PListBuddy`.
5. `_scripts/stow.sh` - Clones the repo and Stows our dotfiles, uses `stow` (nothing special here either).
6. `install.sh` - Combines all the scripts mentioned above.

### _scripts/brew.sh
There are two main functions in this script and they do the same thing, the only difference it the command. `install_packages()` uses `brew install` then `install_casks()` uses `brew install --cask`.

This function takes an array of brew formulas. `$@` is a special variable in Bash that represents all the arguments passed to the function. Wrapping it in parentheses (("$@")) converts these arguments into an array.

We loop through every item in the array and call the `brew list` command on each item.
* `/dev/null` is a special file in Unix-like systems that discards any data written to it. Think of it as a “black hole” for unwanted output.
* `&> /dev/null` redirects both standard output (stdout) and standard error (stderr) to /dev/null, effectively suppressing all output from the command.

The reason we are suppressing the output is because we don’t really care about it, we only want to test the exit status of the command `brew list`, if the package is installed the exit status is 0 (success) otherwise its 1 (failure). The `!` in front of the if statement is to negate the state because we don’t want to install the package of the exit status is 0.
```bash
install_packages() {
  local packages=("$@")
  for package in "${packages[@]}"; do
    if ! brew list "$package" &> /dev/null; then
      echo "Installing $package..."
      brew install "$package"
    else
      echo "$package is already installed."
    fi
  done
}
```

This is how we’d use it.
```bash
# List of packages to install
packages=(
  stow
  git
  git-delta
  fzf
  tldr
  tmux
  jq
  ripgrep
  mas  # mac app store
  # Add more packages here
)

# Run the functions with the lists as arguments
install_packages "${packages[@]}"
}
```

### _scripts/appstore.sh
This is very similar to `brew.sh`, only difference is we are using `mas` CLI instead of brew.
This also takes an Array but we are splitting each item
* **appid**: Uses `"${app%%:*}"` to extract the portion of the app string before the colon (:)
	* `%%` is a Bash string manipulation operator that removes the longest match of the pattern `:*` from the end of the string. Here, it effectively strips everything from the first colon to the end, leaving just the app ID.
* **appname**: Uses `"${app##*:}"` to extract the portion of the app string after the colon (:)
	* `##` is another Bash string manipulation operator that removes the longest match of the pattern `*:` from the beginning of the string. Here, it effectively strips everything up to and including the first colon, leaving just the app name.

```bash
install_from_appstore() {
    local apps=("$@")  # Accept the app list as arguments

    for app in "${apps[@]}"; do
        local appid="${app%%:*}"
        local appname="${app##*:}"

        # Check if the app is already installed
        if mas list | grep -q "$appid"; then
            echo "$appname is already installed."
        else
            echo "Installing $appname..."
            mas install "$appid"
        fi
    done
}
```

This is how we use it
```bash
app_list=(
    "302584613:Kindle"
    "1018301773:AdBlock Pro"
    "441258766:Magnet"
)

install_from_appstore "${app_list[@]}"
```
### _scripts/app_settings.sh
This script uses both `defaults` and `PListBuddy`, I created a function for `PListBuddy` that basically checks if the config we want to set exists in the `.plist` file, if not then we'll add it.
```bash
# Function to set or add plist properties
set_or_add_plist_property() {
    property=$1
    type=$2
    value=$3
    plist_path=~/Library/Preferences/com.googlecode.iterm2.plist

    # Check if the property exists, suppressing both stdout and stderr
    /usr/libexec/PlistBuddy -c "Print $property" "$plist_path" > /dev/null 2>&1

    if [ $? -eq 0 ]; then
        # If property exists, set it, suppressing both stdout and stderr
        /usr/libexec/PlistBuddy -c "Set $property $value" "$plist_path" > /dev/null 2>&1
        new_value=$(/usr/libexec/PlistBuddy -c "Print $property" "$plist_path")
        echo "Set: $property -> $new_value"
    else
        # If property does not exist, add it, suppressing both stdout and stderr
        /usr/libexec/PlistBuddy -c "Add $property $type $value" "$plist_path" > /dev/null 2>&1
        new_value=$(/usr/libexec/PlistBuddy -c "Print $property" "$plist_path")
        echo "Add: $property -> $new_value"
    fi
}
```
Then the rest of the script is just a bunch of `defaults` commands.

## Installation
To install my setup, I recommend forking my repo and updating the scripts `_scripts/*.sh`, add or remove plugins you don’t need.
Update the config files in each folder and delete any you don’t require. If you have more `.doffile` to `stow` then make sure you update the `_scripts/stow.sh`, I have a map variable called `dotfiles_list`
```bash
# Stow Map folder_name:file_name
dotfiles_list=(
  "zsh:.zshrc"
  "p10k:.p10k.zsh"
  "hushlogin:.hushlogin"
  "git:.gitconfig"
)
```

A useful trick is to get the raw file of the installation script from GitHub by pressing the “Raw” button on your file, and then use a curl command to fetch it:
```bash
curl -fsSL https://raw.githubusercontent.com/DarrenVictoriano/dotfiles/terminal_setup/install.sh
```
here’s what the flag means:
1. **-f (or --fail)**: Fail silently on server errors. If the HTTP server responds with a 4xx or 5xx error, curl will not output the response body and will return a non-zero exit status. This is useful to avoid inadvertently executing incomplete or erroneous content.
2. **-s (or --silent)**: Silent mode. This flag tells curl to operate quietly and suppress progress bars, error messages, and other output. This is often used in scripts to avoid cluttering the output.
3. **-S (or --show-error)**: Show errors. When used with -s (silent mode), this flag ensures that errors are still shown if curl encounters one. It helps by providing error messages without the usual progress output.
4. **-L (or --location)**: Follow redirects. This flag tells curl to follow any HTTP 3xx redirect responses. It’s helpful when the URL you are trying to fetch is redirected to another location.

Then we can pass this as an argument to `sh` and use the flag `-c` that will allow it to execute any string argument as command.
So this is how we install it:

1. Assuming you are starting from a fresh Mac, open your native terminal app
2. Install Homebrew and make sure to follow the instructions to add it to your $PATH
3. Run this command:
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/DarrenVictoriano/dotfiles/terminal_setup/install.sh)"
```

## Conclusion
By using tools like GNU Stow, you can simplify and automate your terminal setup, making it easy to replicate your preferred environment across multiple machines. This not only saves time but also ensures consistency in your workflow. If you’re interested in using my setup, feel free to fork my repository, customize the scripts to suit your needs, and follow the installation steps provided.

I hope this guide helps you streamline your terminal configuration as much as it helped me and enjoy a more efficient and personalized development environment. If you have any questions or suggestions, feel free to reach out or leave a comment below.

Happy coding!