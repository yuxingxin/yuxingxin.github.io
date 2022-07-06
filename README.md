## MyBlog

Deviations from [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy/)

## Building / Testing Locally

On Ubuntu / Intel-based Mac:

```sh
bundle install
bundle exec jekyll build
bundle exec jekyll s --incremental --watch
```

On Apple M1 chip:

```sh
arch -arch x86_64 bundle install
arch -arch x86_64 bundle exec jekyll build
arch -arch x86_64 bundle exec jekyll s --incremental --watch


# if still having issues with ffi, also run:
arch -x86_64 sudo gem install ffi
```