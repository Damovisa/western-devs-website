---
layout: post
title:  "Why Gulp?"
date: 2015-09-02T18:41:57-04:00
categories:
comments: true
authorId: dave_paquette
originalurl: http://www.davepaquette.com/archive/2015/09/01/why-gulp.aspx
---

I recently made some updates to my blog post on [How to Use Gulp in Visual Studio][1]. I don't usually go back and update old blog posts, but this one receives a fair amount of daily traffic. There was a minor mistake in the way I had setup my gulp watch and I wanted to fix that to avoid confusion. I also get a lot of questions about why using a task runner like Gulp is a 'better approach' than the way things are done in ASP.NET 4.x. I have addressed some of those questions in the original post but I will go into more detail here.

<!--more-->

Let's start with a quick example using the 2 approaches.

### System.Web.Optimization

In previous versions of ASP.NET, optimizations such as bundling and minification are done using the System.Web.Optimization package. In this approach, we configure our bundles in C#:

{% codeblock lang:csharp %}
public class BundleConfig
{
    // For more information on bundling, visit http://go.microsoft.com/fwlink/?LinkId=301862
    public static void RegisterBundles(BundleCollection bundles)
    {
 
        bundles.Add(new ScriptBundle("~/bundles/js").Include(
                    "~/app/Script1.js",
                    "~/app/Script2.js"));
 
    }
}
{% endcodeblock %}

Those bundles are referenced in our Razor views as follows:

    @Scripts.Render("~/bundles/js")

When running in Release mode, the server combines the files in a bundle into a single minified file and renders a single &lt;link&gt; or &lt;script&gt; tag for the bundle. When running in Debug mode, the server renders individual &lt;link&gt; or &lt;script&gt; tags for each file in the bundle. The file optimization step is done at runtime. A version hash is added to the bundle URL to support aggressively caching the asset on the client side.

#### Task Runners

When using a Task Runner like Gulp (or Grunt), optimizations like bundling and minification are done at build/compile time. The bundles and any step related to bundling are configured in a JavaScript file that is executed by the task runner. Here is a simple example of a gulp file that does the same optimizations as the example above:

{% codeblock lang:javascript %}
// include plug-ins
var gulp = require('gulp');
var concat = require('gulp-concat');
var uglify = require('gulp-uglify');
 
var config = {
    //Include all js files but exclude any min.js files
    src: ['app/**/*.js', '!app/**/*.min.js']
}
 
gulp.task('scripts', function () {
 
    return gulp.src(config.src)
      .pipe(uglify())
      .pipe(concat('all.min.js'))
      .pipe(gulp.dest('app/'));
});
 
//Set a default tasks
gulp.task('default', ['scripts'], function () { });
{% endcodeblock %}

_Note that this is a simplified example. For a more complete example see my [original post][6]._

By running the scripts task, all the JS files in my app folder are combined and minified into a single all.min.js file. In ASP.NET 5, we can decided based on our current environment if we should include references to the individual files or the single combined and minified file.

{% codeblock lang:xml %}
<environment names="Development">
    <script asp-src-include="~/app/**/*.js" asp-src-exclude="~/app/**/*.min.js"></script>
</environment>
<environment names="Staging,Production">
    <script src="~/app/all.min.js" asp-append-version="true"></script>
</environment>
{% endcodeblock %}

In this case, the files are combined and minified at build/compile time. The minified version of the file is published to the server. At runtime, Razor tag helpers are responsible for [deciding which script tags][2] to include. The tag helpers also [append the file version hash][3] to support aggressively caching the files on the client side. As was covered in my original post, we can use the Task Runner Explorer to link the Scripts task to the build event in Visual Studio. Using a watch, I can automatically run the Scripts task anytime a JS file changes.

## Why I prefer the Task Runner approach

Now let's get into the details of why I prefer using a task runner like Gulp over the runtime optimization approach taken by System.Web.Optimization.

### Runtime vs. Compile-Time Optimizations

System.Web.Optimization takes the approach of bundling/minifying your assets at runtime. The first time a request comes in for a bundle, it will combine and minify all the files in that bundle and cache the results for the next request. While the cost of this is minimal, it has always seemed to me that it is a strange to use server resources to do this task. At the time of publishing our application to the server, we already know what the code is. To me it makes more sense to do this step on the build server or on the developer machine BEFORE publishing the application. Task runners like Gulp take the approach of doing these asset optimization steps at compile/build time.

This becomes a bigger advantage when we start doing more than just bundling and minification. My typical _scripts_ task takes all theTypeScript files from my app, compiles them to JavaScript, combines the output of that to a single minified JS file and writes out source maps. Gulp allows me to easily automate all of this with a single task. Compiling TypeScript and generating source maps is just not possible with System.Web.Optimization and I don't think anyone would argue that doing all those steps on the web server at runtime would make sense anyway. Yes, some of these steps could be handled using Visual Studio plugins…more on that later.

For the vast majority of applications, I think the task runner approach is more logical. You are shipping known, pre-optimized assets to your production server. Don't make your server do more than it needs to.

_Note that there are some specific use cases such as CMS tools that require runtime optimizations because the assets might not be known at compile time._

### Extensibility and Consistency

There is no question that the runtime bundling in MVC 5 provides a better 'out-of-the-box' experience. When you create a new project, bundling and minification is setup and working. It is easy to add new files. People generally understand the concepts and don't need to spend a lot of time fiddling with the bundle configuration. As I have eluded to in the TypeScript example, System.Web.Optimization starts to fall apart for me is when you want to take things 1 step further.

Let's consider another example. What if I want to start using a CSS pre-processor like LESS or SASS? There is no way built-in way to tie CSS pre-processors into System.Web.Optimization. Now you need to start looking for VS plugins to do this task. If we're lucky, these will work well. In my experience they have some problems, are often out-of-date or are just not available. One big problem with using VS plugins is that I can't make use of those on the build server which means I now need to check my generated CSS files in source control. I much prefer to only check in my LESS or SASS source files and have the build server generate the CSS files. (Checking in generated files pollutes the commit logs and makes code reviews a lot less effective).

Another problem is trying to make sure that everyone on the team has the right plugins installed. There are ways to enforce this, but it is not very easy.

With Gulp, all we need to do is include a gulp plugin (eg, gulp-less) and add the less compilation step to my stylesheet task. It is a 1 or 2 line change to my gulp file. The node package manager is able to ensure that everyone on the team has the right gulp plugins installed. Since everything is command line based, it is also very easy to [call the same tasks from the build server][4].

So the big advantages that I see are extensibility and consistency. System.Web.Optimization is very good at doing a couple things, but it is also limited to doing those couple of things. When we want to take things a little further, we start to run into some pain points with ensuring a consistent development environment. Gulp on the other hand is extremely flexible and extensible in a way that makes it easy to provide consistency environment and consistent builds across your entire team.

## Wrapping it up

In small and simple MVC 5 projects, I still use System.Web.Optimization for it's simplicity. For more complex projects where I want to use some newer web dev tooling, I use Gulp. Gulp gives me a lot more options and the opportunity to design a better workflow for my team.

The File-New Project experience in current beta version of MVC 6 uses Gulp. I'm excited about this, but the default gulp file is in need of some work. It is difficult to extend and contains some errors that will cause problems for those who are new to Gulp. Of course, this is a beta version and the team is still working on this. I am hopeful that the experience will improve before the official release of MVC 6. In the meantime, don't be afraid to learn about Gulp and all the amazing things it can do. I find the [Gulp Recipes][5] to be a very valuable learning tool.

[1]: http://www.davepaquette.com/archive/2014/10/08/how-to-use-gulp-in-visual-studio.aspx
[2]: http://www.davepaquette.com/archive/2015/05/05/web-optimization-development-and-production-in-asp-net-mvc6.aspx
[3]: http://www.davepaquette.com/archive/2015/05/06/link-and-script-tag-helpers-in-mvc6.aspx
[4]: http://www.davepaquette.com/archive/2015/04/08/integrating-gulp-and-bower-with-visual-studio-online-hosted-builds.aspx
[5]: https://github.com/gulpjs/gulp/tree/master/docs/recipes
[6]: http://www.davepaquette.com/archive/2014/10/08/how-to-use-gulp-in-visual-studio.aspx

