---
title: 一个自用的gulpfile配置
date: 2017-08-14 21:21:17
tags: javascript Tools
---

## 模块介绍

### node模块介绍：
- fs：内置的核心模块，用到他提供的`fs.accessSync`方法，可判断文件是否存在

<!--more-->

### 其他模块：
- `argv = require('yargs').argv` 获取命令参数，如执行`gulp build --prod`。通过production = !!argv.prod可判断执行命令带了这个参数。
- gulp-less, gulp-concat, gulp-rename, gulp-notify, gulp-sourcemaps 不解释
- gulp-uglify 用来压缩JS, 
- gulp-if 在这里如果是product环境执行sourcemap. `.pipe(gulpif(!production, sourcemaps.init()))`
- [gulpSequence](https://github.com/teambition/gulp-sequence) 确保执行顺序。

## 工作流程
我是把 js 和 css 都分为两种，vendor 和 app。vendor 是第三方的，app 是自己写的。要分别打包。
执行`gulp`，默认执行的是`gulp default`。进而执行 build task。
其内容是 `gulp.task('build', gulpSequence('clean', ['JS:vendor', 'JS:app', 'CSS:vendor', 'CSS:app'], 'rev'));`
先清理老的生成文件，然后打包 js 和 css ，这几个task可以平行进行，最后执行rev。用来生成rev-manifest.json。

## 关于manifest.json
这个地方是最头疼的。目前的方案并不完美。
`rev`task目的是生成 rev-manifest.json 文件。内容类似：
```
{
  "app.js": "app-05bf83e6b8.js",
  "app.min.css": "app-da1387dd51.min.css",
  "vendor.js": "vendor-54f376abce.js",
  "vendor.min.css": "vendor-3700dac008.min.css"
}
```

然后我写了个PHP方法

```php
public static function assets($file, $buildDirectory = 'dist') {
    static $manifest;
    static $manifestPath;

    if (is_null($manifest) || $manifestPath !== $buildDirectory) {
        $manifest = json_decode(file_get_contents(($buildDirectory.'/rev-manifest.json')), true);

        $manifestPath = $buildDirectory;
    }

    if (isset($manifest[$file])) {
        return '/'.$buildDirectory.'/'.$manifest[$file];
    }

    throw new ErrorException("File {$file} not defined in asset manifest.");

    //throw new InvalidArgumentException("File {$file} not defined in asset manifest.");
}
```

在layout模版文件中
调用这个方法，类似：
`<link rel="stylesheet" href="<?= Common::assets('vendor.min.css')?>">`
这样返回的结果就是`<link rel="stylesheet" href="vendor-3700dac008.min.css>`


### 为什么不动态生成模版文件?
gulpfile中有一个被弃用的名为version的task。使用revCollector模块可以生成模版文件，尝试后并不好用。
还有一点是每次修改 css 或 js 都要重新生成，map hash肯定也会改变。进而rev-manifest.json要重新生成。
但是我发现，比如我改了一处样式。
原来的是：
```
{
  "app.js": "app-05bf83e6b8.js",
  "app.min.css": "app-da1387dd51.min.css",
  "vendor.js": "vendor-54f376abce.js",
  "vendor.min.css": "vendor-3700dac008.min.css"
}
```
我希望重新生成后是
```
{
  "app.js": "app-05bf83e6b8.js",
  "app.min.css": "app-xxxxxx.min.css",
  "vendor.js": "vendor-54f376abce.js",
  "vendor.min.css": "vendor-3700dac008.min.css"
}
```
但结果却是
```
{
  "app.min.css": "app-xxxxxx.min.css"
}
```
虽然我看文档使用了`.pipe(rev.manifest({merge: true}))`。
但不生效，所以我只好曲线救国，写了个`manifest:merge`的task。


## 其他缺点：
因为没有使用browser-sync，即便gulp watch。修改样式后照样要手动刷新浏览器。
没有eslint

## 总结
对于比较小的项目，使用gulp还是很方便的。看着比较清晰。
还有一个小技巧是，如果你可以[github gist](https://gist.github.com/search?utf8=%E2%9C%93&q=gulpfile+stars%3A%3E100&ref=searchresults)搜下别人的gulpfile都是怎么写的。能获得不少灵感。

```javascript
const fs = require('fs');

require('shelljs/global');

const gulp       = require('gulp'),
    less         = require('gulp-less'),
    uglify       = require('gulp-uglify'),
    concat       = require('gulp-concat'),
    rename       = require('gulp-rename'),
    rev          = require('gulp-rev'),
    gulpSequence = require('gulp-sequence'),
    revCollector = require('gulp-rev-collector'),
    sourcemaps   = require('gulp-sourcemaps'),
    notify       = require('gulp-notify'),
    argv         = require('yargs').argv,
    gulpif       = require('gulp-if'),
    extend       = require('gulp-extend'),
    es           = require('event-stream'),
    autoprefixer = require('gulp-autoprefixer');

const production = !!argv.prod;


// gulp (watch) : for development
// gulp build : for a one off development build
// gulp build --prod : for a minified production build

const Asset = {
    vendorJS: {
        sourcePath  : 'frontend/web/src/js/vendor/',
        // 注意不用带.js
        sourcefiles : [
            'jquery.min',
            'bootstrap.min',
            'angular.min',
			// ...
        ],
        distFileName: 'vendor.js'
    },

    appJS: {
        sourcePath  : 'frontend/web/src/js/app/',
        sourcefiles : [
            'common.js',
        ],
        distFileName: 'app.js'
    },

    vendorCSS: {
        sourcePath  : 'frontend/web/src/styles/vendor/',
        sourcefiles : [
            'bootstrap.min.css',
            // ...
        ],
        distFileName: 'vendor.css'
    },

    appCSS: {
        sourcePath  : '',
        sourcefiles : [
            'frontend/web/less/main.less',
        ],
        distFileName: 'app.css'
    },

    cleanSrc: [
        'frontend/web/dist/*',
    ],
    dest    : 'frontend/web/dist/',
};

function fsExistsSync(path) {
    try{
        fs.accessSync(path, fs.F_OK);
    } catch(e){
        return false;
    }
    return true;
}

function srcFilePath(path, files, suffix='') {
    return files.map(function (file) {
        let fullPathFile = path + file + suffix;
        if (fsExistsSync(fullPathFile)) {
            return fullPathFile;
        }else if (fsExistsSync(path + file + '.js')){
            return path + file + '.js';
        }else {
            console.log('no file error:' + files);
        }
    });
}

// concat all vendor js to vendor.js
gulp.task('JS:vendor', function () {
    const config = Asset.vendorJS;
    const files = srcFilePath(config.sourcePath, config.sourcefiles, production ? '.min.js' : '.js');
    return gulp.src(files)
        .pipe(concat(config.distFileName))
        .pipe(gulp.dest(Asset.dest + 'js/'))
    //  .pipe(notify({message: 'generated ' + config.distFileName}));
});

gulp.task('JS:app', function () {
    const config = Asset.appJS;
    const files = srcFilePath(config.sourcePath, config.sourcefiles);
    return gulp.src(files)
        .pipe(concat(config.distFileName))
        .pipe(gulp.dest(Asset.dest + 'js/'))
    //  .pipe(notify({message: 'generated ' + config.distFileName}));
});

// concat all vendor css files into vendor.css
gulp.task('CSS:vendor', function () {
    const config = Asset.vendorCSS;
    const files = srcFilePath(config.sourcePath, config.sourcefiles);
    return gulp.src(files)
        .pipe(sourcemaps.init())
        .pipe(concat(config.distFileName))
        .pipe(rename('vendor.min.css'))
        .pipe(sourcemaps.write('./'))
        .pipe(gulp.dest(Asset.dest + 'css/'))
    // .pipe(notify({message: 'generated ' + config.distFileName}));
});

gulp.task('CSS:app', function () {
    return gulp.src(Asset.appCSS.sourcefiles)
        .pipe(sourcemaps.init())
        .pipe(less({compress: true}))
        .pipe(autoprefixer())
        .pipe(rename('app.min.css'))
        .pipe(sourcemaps.write('./'))
        .pipe(gulp.dest(Asset.dest + 'css/'))
    // .pipe(notify({message: 'generated ' + config.distFileName}));
});


// delete old files before generate
gulp.task('clean', function () {
    rm('-rf', Asset.cleanSrc);
});

gulp.task('rev', ['copyMap'], function () {
    return gulp.src([Asset.dest + 'js/' + '*.js', Asset.dest + 'css/' + '*.css'])
        .pipe(rev())
        .pipe(gulp.dest(Asset.dest))
        // generate rev-manifest.json
        .pipe(rev.manifest({merge: true}))
        .pipe(gulp.dest(Asset.dest));
});

gulp.task('copyMap', function () {
    return gulp.src([Asset.dest + 'css/' + '*.map', Asset.vendorJS.sourcePath + '*.map'])
        .pipe(gulp.dest(Asset.dest));
});

//  弃用
gulp.task('version', function () {
    // read rev-manifest.json and replace the mapped files
    return gulp.src([Asset.dest + 'rev-manifest.json', 'frontend/web/src/*.php'])
        .pipe(revCollector({
            replaceReved: true,
        }))
        .pipe(gulp.dest('frontend/views/version/'));
});

gulp.task('build', gulpSequence('clean', ['JS:vendor', 'JS:app', 'CSS:vendor', 'CSS:app'], 'rev'));

gulp.task('default', ['build']);

// ---- watch tasks  ----

gulp.task('watch:clean', function () {
    rm('-rf', 'frontend/web/dist/app*.css');
    rm('-rf', 'frontend/web/dist/app*.map');
});

gulp.task('manifest:rebuild', function () {
    return gulp.src(Asset.appCSS.sourcefiles)
        .pipe(sourcemaps.init())
        .pipe(less({compress: true}))
        .pipe(autoprefixer())
        .pipe(rename('app.min.css'))
        .pipe(sourcemaps.write('./'))
        .pipe(rev())
        .pipe(gulp.dest(Asset.dest))
        .pipe(rev.manifest('dist/css-rev-manifest.json', {merge: true, base: 'dist'}))
        .pipe(gulp.dest(Asset.dest));
});

gulp.task('manifest:merge', function () {
    let older = gulp.src('frontend/web/dist/rev-manifest.json');
    let newer = gulp.src('frontend/web/dist/css-rev-manifest.json');

    return es.merge(older, newer)
        .pipe(extend('rev-manifest.json'))
        .pipe(gulp.dest(Asset.dest));
});


gulp.task('watch', function (event) {
    gulp.watch('frontend/web/less/*.less', function (event) {
        gulpSequence('watch:clean', 'manifest:rebuild', 'manifest:merge')(function (err) {
            if (err) console.log(err)
        })
    });
});
```

