# Blog

* Built with Jekyll
* Use Langom theme (based on http://mdswanson.com)

## Usage

### First installation

```shell
git clone git@github.com:kdisneur/blog.git
cd blog
git submodule update
bundle
```

### Development

```shell
jekyll serve -w
```

### Deployment

```shell
jekyll build
cd build
git add .
git commit
git push
cd ..
```
