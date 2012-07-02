---
layout: post
title: "Building a real application with BackboneJS"
date: 2012-07-02 16:12
comments: true
category:  BackboneJS
---

A couple of months ago I began looking at new things to learn and BackboneJS caught my eye. So I started looking for articles and tutorials however I quickly realised that they only covered very simple concepts(AKA to-do apps). So I began this project with the goal of creating a more full featured application.

The application I choose to create was a jsFiddle style application  that includes the ability to create multiple files.  However the point of this article isn’t just to talk about why I made this application but to share with you what I learnt.

[Application](https://webide.co/) | [Source](https://github.com/dancannon/WebIDE/)

<!--More -->

## Application Architecture
For this application I used Tim Branyens backbone.boilerplate and backbone.layoutmanager along with RequireJS. I also used backbone.relational to deal with associations between models.  The use of backbone.boilerplate and RequireJS allowed me to have structure the javascript side of the application in a straight forward way which I believe is one of the hardest parts of backbone. However even after writing this application I don’t believe that RequireJS is the answer to our javascript loading issues. Sure it solves some problems such as dependency management however the fact that many libraries do not offer support so you end up having to deal with shim plug-ins and a load more configuration for these files. Another issue I have with it is that you have to include even more code to run the code, in my ideal world the compiled code would only contain the application code.

For the back-end I used the PHP framework Symfony2 with the FOSRestBundle to create a RESTful API. Although this framework is not as lightweight as some solutions it allowed me to quickly set-up the database access and the large collection of "bundles" meant that I didn’t need to reinvent the wheel with things like user authentication.

## Modularity
Modularity is a key part of any application so it was important to ensure that all parts of this application were as modular as possible. Backbone boilerplate and RequireJS help this by keeping modules in separate files and managing dependencies however for a large scale application the most important thing is events. Backbone has a powerful event system which should be used to send events between different modules using the name-spaced app object. Here is an example from the application

```javascript Trigger an event https://github.com/dancannon/WebIDE/blob/master/src/WebIDE/SiteBundle/Resources/public_src/js/app/modules/header.js#L50 Source
save: function() {
    app.trigger("save");
},
```

```javascript Event callback https://github.com/dancannon/WebIDE/blob/master/src/WebIDE/SiteBundle/Resources/public_src/js/app/modules/project.js#L45 Source
app.on("save", function() {
    that.save().success(function() {
        if (that.has('current_version')) {
            app.router.navigate('/' + that.id + '/' + that.get('current_version'));
        }

        app.trigger("reload");
    });
});
```

## Authentication
Authentication is handled on the server side by [FriendsOnSymfony's](https://github.com/FriendsOfSymfony/FOSRestBundle) Symfony2 bundle FOSUserBundle. As mentioned before the use of these bundles reduced development time. However this bundle only dealt with security on the server side so I had to come up with another way of dealing with it on the client side. To do this I choose to send different HTTP status codes when requesting something from the API. For example if an anonymous user tries to send a POST request to /projects then a 503 error code will be returned. Alternatively if a user tries to edit a project they do not have access to by sending a PUT request to /projects they a 501 error code will be returned.

``` php Server side authentication code https://github.com/dancannon/WebIDE/blob/master/src/WebIDE/SiteBundle/Controller/ProjectController.php#L291 Source
<?php
protected function checkPermission(OwnableEntity $entity=null)
{
    if($entity !== null) {
        if(!$this->get('security.context')->isGranted('ROLE_USER')) {
            throw new HttpException(401, "Unauthorized");
        }
        if($this->get('security.context')->getToken()->getUser() !== $entity->getUser()) {
            throw new HttpException(403, "Unauthorized");
        }

        return true;
    } else {
        if(!$this->get('security.context')->isGranted('ROLE_USER')) {
            throw new HttpException(401, "Unauthorized");
        }

        return true;
    }
}
?>
```

On the client side we then check if the response code is 200, if it is not the we show an error message to the user.

``` javascript Ajax error checking https://github.com/dancannon/WebIDE/blob/master/src/WebIDE/SiteBundle/Resources/public_src/js/app/main.js#L148 Source
$.ajaxError(function(e, xhr, settings) {
	if(xhr.status === 401) {
		//Not logged in
	}
	if(xhr.status === 403) {
		//Not allowed
	}
}
```

## Templating/Handlebars
For the templating I used handlebars, one of the reasons for this is that it is similar to twig which is used on the server side and it is very simple, however this simplicity is also a slight disadvantage. It would have been nice to have a little more power to allow template inheritance as well as more integration with backbone.

To load templates the application first the cache and if it is not found it then loads the template from the server using ajax. It then caches the template and returns the content. For the cache I choose to use session storage with a fallback to a javascript object. 

The reason that I choose to cache the content instead of including the templates with the rest of the javascript files is that this way the templates only have to be loaded once however in my testing the differences in page loads were negligible.