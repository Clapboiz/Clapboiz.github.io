# Clapboiz.github.io
This is my blog website (Clapboiz.github.io)

### Note: if you are using this repo and now get a notification about a security vulnerability, delete the Gemfile.lock file. 

# Instructions
## To run locally (not on GitHub Pages, to serve on your own computer)

1. Clone the repository and made updates as detailed above
1. Make sure you have ruby-dev, bundler, and nodejs installed: `sudo apt install ruby-dev ruby-bundler nodejs` (any OS)
1. Run `bundle clean` to clean up the directory (no need to run `--force`) [Optional]
1. Run `bundle install` to install ruby dependencies. If you get errors, delete Gemfile.lock and try again.
1. **Instead of using this command `jekyll serve --livereload` to run local web**
1. Not run `bundle exec jekyll liveserve` to generate the HTML and serve it from `localhost:4000` the local server will automatically rebuild and refresh the pages on change. (Outdated versions and it doesn't work)

**You can use `run.bat` instead of `jekyll serve --livereload`**

```
.\run.bat (powershell)
  or
run.bat (command prompt)
```

If you get an error, please look at the gemfile I configured, and if you follow it, it will run

Please follow the 2 links below if you need to see how to solve the problem
```
https://github.com/tzinfo/tzinfo/wiki/Resolving-TZInfo::DataSourceNotFound-Errors

https://github.com/tzinfo/tzinfo/issues/144
```
### Note: Do not format html files, because the format will be corrupted (Because when formatting html files, spaces will appear, causing url and icon errors).

# REFERENCE
[1] https://github.com/academicpages/academicpages.github.io
