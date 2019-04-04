<!--
 * @Description: 
 * @Author: Damon.chen
 * @LastEditors: Damon.chen
 * @Date: 2019-04-04 11:25:41
 * @LastEditTime: 2019-04-04 11:30:41
 -->
1. 首先确认有node的环境，没有就装一个
    
2. 全局安装 gulp：
    ```
    npm install --global gulp
     ```   
3. 作为项目的开发依赖安装：
    ```
    npm install --save-dev gulp
    ```
4. 安装所需的各种插件（配置文件种说明了各个插件的用途）：
```
npm install jshint gulp-jshint gulp-sass gulp-concat gulp-uglify gulp-rename imagemin --save-dev
```
    
5. 在项目根目录下创建一个名为 gulpfile.js 的文件，文件配置如下:


```
   // 引入 gulp
    var gulp = require('gulp'); 
    
    // 引入组件
    var jshint = require('gulp-jshint'); //js代码校验 
    var sass = require('gulp-sass');    //编译css
    var imagemin = require('gulp-imagemin'); //压缩图片
    var concat = require('gulp-concat');
    var uglify = require('gulp-uglify');
    var rename = require('gulp-rename');    
    
    // 检查脚本
    gulp.task('lint', function() {
        gulp.src('./static/src/js/*.js')
            .pipe(jshint())
            .pipe(jshint.reporter('default'));
    });
    
    // 编译Sass
    gulp.task('sass', function() {
        gulp.src('./static/src/scss/*.scss')
            .pipe(sass())
            .pipe(gulp.dest('./static/dist/css'));
    });
    
    // 压缩图片
    gulp.task('imagemin', function() {
        gulp.src('./static/src/images/*.{png,jpg,gif,ico}')
             .pipe(imagemin({
                optimizationLevel: 5, //类型：Number  默认：3  取值范围：0-7（优化等级）
                progressive: true, //类型：Boolean 默认：false 无损压缩jpg图片
             }))
             .pipe(gulp.dest('./static/dist/images'));
     });
    // 合并，压缩js文件
    gulp.task('scripts', function() {
        gulp.src('./static/src/js/*.js')
            .pipe(concat('all.js'))
            .pipe(gulp.dest('./static/dist/js'))
            .pipe(rename('all.min.js'))
            .pipe(uglify())
            .pipe(gulp.dest('./static/dist/js'));
    });
    
    // 默认任务
    gulp.task('default', function(){
        gulp.run('lint', 'sass','imagemin', 'scripts');
    
        // 监听文件变化
        gulp.watch([
            './static/src/scss/*.scss',
            './static/src/images/**',
            './static/src/js/*.js'
        ], function(){
            gulp.run('lint', 'sass','imagemin', 'scripts');
        });
    });
    
    ```

