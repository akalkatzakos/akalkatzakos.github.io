---
layout: post
published: true
title: SPA with Spring Boot, Bootstrap and AngularJS 
---
## Intoduction
SPA (Single Page Applications) have never been easier to build with technologies like Spring Boot, AngularJS and Bootstrap. 
The idea of SPA applications resolves around the idea of a main page which gets loaded once and then everything is done through Ajax calls asynchronously and without relaoding of the page.  
The generic architecture of an SPA is composed of two separate parts, frontend and backend, a standard JSON API to exchange information over HTTP and the frontend being totally responsible for everything that the uses sees.  
In this sense backend is now responsible only to provide the required Json payloads responding to HTTP verbs (GET, POST, PUT, DELETE) and doesn't build any presentation layer pages (like JSP and JSF do). What is more the frontend accomplishes everything without refreshing the whole page but only particular pieces in the page using of course AJAX.  
Benefits of this way of building SPA are the following:

1. Separate teams having backend and frontend as only a definition of the JSON API is required and both can work independently from then on.
+ Much easier testing of the front or the back end
+ Much easier integration of other clients (f.e. mobile app) consuming the same JSON Api therefore allowing you easy expansion to other devices

In this post we are going to build a simple SPA with CRUD operations for a simple Bookmarks application where the user can view/edit/save/delete bookmarks grouped in categories. For the backend part we will use Spring Boot and Mongodb and for the front end we will use AngularJS and Bootstrap.  
For demonstration purposes and in order to be able to cover more things related to AngulaJS the application's page has 2 tabs.

* One contains the form to add/edit a new Bookmark:

<img src="https://raw.githubusercontent.com/akalkatzakos/akalkatzakos.github.io/master/_images/20151120/form.png" style="width: 350px;" />

+ and the other displays the big overview:

![Bookmarks list][bookmarks_list]

---

## Backend
### Setup
Starting with the Backeend whose sole responsibility is to reply to Http requests with JSON payload we will utilise the Spring Boot framework which makes all the setup very easy. Opening [**Start Spring IO**] [start_spring_io] we are in a web application where we can define what we need for our project and generate at the end a zip file which we can use as the skeleton of our project. We need only __Web__ and __MongoDb__.  
After we extract the project we are good to go. One last setup test is to start on a directory of our choice the mongoDB by running _mongod -dbpath database/_

### Model
Our model is very simple, just a Pojo representing the Bookmark entity we want to work with.

```Java

import org.springframework.data.annotation.Id;

public class Bookmark {

	private String name;
	private String url;
	private String folder;
	
	@Id
	private String id;
	
        //getters/setters	

}


```

### Logic
Our codebase already contains a generated class ending in Application which is annotated with _SpringBootApplication_ and it is basically the one which will be used to start our application. Because of the annotation also it scans and wires up all Spring object we declare like Controllers, Services etc...    

First we declare a repository interface  _BookmarkRepository_ which extends _MongoRepository<Bookmark, String>_ and will basically cause Spring Data to implement besides the common CRUD operations, any other methods that we have defined inside our repository. For this case we just define a method to give us back the bookmarks for a folder.  

We proceed  with defining our _Controller_ class which will contain all of the request/response processing. This class needs to be annotated with _RestController, RequestMapping("/bookmarks")_  where _/bookmarks_ is the url endpoint we will need to access our service.  

A GET and a POST operation will be shown here as the rest follow the same coding pattern.  

```Java
	@Inject
	private BookmarkRepository bookmarkRepository;

	@RequestMapping(method = RequestMethod.GET)
	public ResponseEntity<Iterable<Bookmark>> getAllBookmarks() {
		Iterable<Bookmark> all = bookmarkRepository.findAll();
		return new ResponseEntity<>(all, HttpStatus.OK);
	}

	@RequestMapping(method = RequestMethod.POST)
	public ResponseEntity<?> createBookmark(@RequestBody Bookmark bookmar) {
		Bookmark saved = bookmarkRepository.save(bookmar);

		// Set the location header for the newly created resource
		HttpHeaders responseHeaders = new HttpHeaders();
		URI newBkUri = ServletUriComponentsBuilder.fromCurrentRequest()
				.path("/{id}").buildAndExpand(saved.getId()).toUri();
		responseHeaders.setLocation(newBkUri);

		return new ResponseEntity<>(saved, responseHeaders, HttpStatus.CREATED);
	}

```

After the injection of the repository, we declare a GET method which returns all the bookmarks using the _findAll_ method of the repository. A _ResponseEntity_ object is returned which is SPring way of encapsulating a response body along with a response header. An _OK(200)_ code is retuned in this case. The POST method uses again the injected repository to do the save and then builds a _CREATED(201)_ response with a location header of the url where this link will be available. In our example this will be something of the form _http://localhost:8080/bookmarks/565a1de077c88fe61a7d06b5_. For this request to be served we need yet a other GET request method:

```Java

	@RequestMapping(value = "/{bookmarkId}", method = RequestMethod.GET)
	public ResponseEntity<?> getBookmark(@PathVariable String bookmarkId) {
		Bookmark p = bookmarkRepository.findOne(bookmarkId);
		return new ResponseEntity<>(p, HttpStatus.OK);
	}

```  
Which will return the JSON represting the created bookmark.  
We can easily test our JSON API using a plugin for the browser we are using. For Chrome i use Postman.  
First we create a POST reauest:

![Create a Bookmark][post_create]

Notice in the response headers we got a Location header with the url we can use to retrieve the bookmark back. If we browse to it we get:

![View a Bookmark][view_bookmark]

Using POSTMAN once again for our first method we get back a JSON array of all the bookmarks created so far:

![View Bookmarks][view_bookmarks]

The rest of the methods follow similar patter and define other url endpoints to view particular details needed for the front end.  

## Frontend

###
For our application we will have 2 AngularJS controllers, each one handling each of the tabs we have. The files that compose the frontend pat of the application have the following structure:

![Front End tree structure][front-end-structure]

Our application is composed of:

1. an index.html file which defines the entry point to our app.
+ an app.js file which defines our routes to our modules
+ controllers folder with js files defining the brain of each module
+ partials folder with html files defining the view of our application
+ services folder with common javascript functionality shared betwenn modules
+ css folder with the standard bootstrap css and our custom one
+ js folder with angular javascript files and the ui-bootstap one


 Every request hits this html and particular suffixes in the urls starting with the hash # define the particular parts of our application that we want to visit. This routing is controlled by the app.js file:

```Javascript
bookmarkApp.config(function($routeProvider) {
	$routeProvider.when("/bookmark", {
		controller : 'BookmarkController',
		templateUrl : 'app/partials/bookmark_partial.html'
	}).when("/bookmark/:id", {
		controller : 'BookmarkController',
		templateUrl : 'app/partials/bookmark_partial.html'
	}).when("/list", {
		controller : 'ListBookmarkController',
		templateUrl : 'app/partials/bookmark_list.html'
	}).when("/", {
		redirectTo : "/list"
	});
});
```

This mapping is quite straightforward. As an example if we call url _http://localhost:8080/index.html#/list_
then the ListBookmarkController will execute using as a template the file bookmark_list.html.
The source code for one of the controllers is the following:

```Javascript
// Common pattern when defining a controller 
(function() {

        // Constructor used to wire up Angular objects and services
        // and assign objects and functions to $scope
	function ListBookmarkController($scope, $location, BookmarkProvider, $modal, $log) {
		$scope.listActive = true;
		$scope.messages = [];

                // retrieve folders
		$scope.getFolders = function() {
			BookmarkProvider.getFolders(function(err, folders) {
				if (err) {
					$scope.messages.push({
						type : 'danger',
						msg : err.message
					});
				} else {
					$scope.folders = folders;
				}
			});
		};

		$scope.deleteBookmark = function(bookmark) {
			$scope.toDelete = bookmark;
			$scope.modalInstance = $modal.open({
				animation : true,
				templateUrl : 'deleteTemplate.html',
				scope : $scope,
				size: "sm"
			});

			$scope.modalInstance.result.then(function() {
			}, function() {
				$log.info('Modal dismissed at: ' + new Date());
			});
		};

		$scope.ok = function() {
			BookmarkProvider.deleteBookmark($scope.toDelete, function(err, folders) {
				if (err) {
					$scope.messages.push({
						type : 'danger',
						msg : err.message
					});
				} else {
					$scope.getFolders();
					$scope.getBookmarksInFolder($scope.toDelete.folder);
					$scope.messages.push({
						type : 'success',
						msg : 'Bookmark deleted!'
					});
				}
			});
			$scope.modalInstance.close();
		};

		$scope.cancel = function() {
			$scope.modalInstance.close();
		};

		$scope.closeAlert = function(index) {
			$scope.messages.splice(index, 1);
		};

		$scope.editBookmark = function(bookmark) {
			$location.path('/bookmark/' + bookmark.id);
		};

		$scope.getBookmarksInFolder = function(name) {
			BookmarkProvider.getBookmarksInFolder(name, function(err, bookmarks) {
				if (err) {
					$scope.messages.push({
						type : 'danger',
						msg : err.message
					});
				} else {
					$scope.folder = name;
					$scope.bookmarks = bookmarks;
				}
			});
		};

		$scope.getFolders();
	};

	bookmarkApp.controller('ListBookmarkController', ListBookmarkController);

})();
```

And the template file which this controller uses is:

```html

<div class="container">
	<script type="text/ng-template" id="deleteTemplate.html">
        <div class="modal-header">
            <h3 class="modal-title">Confirm Delete!</h3>
        </div>
        <div class="modal-body">
            Are you sure you want to delete bookmark <span style="color: green; font-weight: bold">{{toDelete.name}}</span>
        </div>
        <div class="modal-footer">
            <button class="btn btn-primary" type="button" ng-click="ok()">OK</button>
            <button class="btn btn-warning" type="button" ng-click="cancel()">Cancel</button>
        </div>
	</script>

	<div class="width300">
		<ul class="nav nav-tabs">
			<li role="presentation" ng-class="{active: listActive}"><a href="#/list">Bookmarks</a></li>
			<li role="presentation" ng-class="{active: bookmarkActive}"><a href="#/bookmark">Add a new Bookmark</a></li>
		</ul>
		<alert ng-repeat="message in messages" type="{{message.type}}" close="closeAlert($index)"> {{message.msg}} </alert>
	</div>


	<h4>Folders</h4>

	<div class="btn-group" role="group">
		<button type="button" ng-repeat="folder in folders" ng-click="getBookmarksInFolder(folder.name)" class="btn btn-default">
			{{ folder.name }} <span class="badge">{{folder.count}}</span>
		</button>
	</div>

	<h4 class="bkm-header" ng-show="bookmarks.length > 0">
		Bookmarks in [<span style="color: green">{{folder}}</span>]
	</h4>
	<!-- 	</div> -->

	<div ng-repeat="bookmark in bookmarks" class="panel panel-default bkmpanel">
		<div class="panel-heading">
			<strong>{{bookmark.name}}</strong>
		</div>
		<div class="panel-body">
			<a href={{bookmark.url}}>{{bookmark.url}}</a>
		</div>
		<div class="floated">
			<button ng-click="deleteBookmark(bookmark)" class="btn btn-danger btn-sm">Delete</button>
			<button ng-click="editBookmark(bookmark)" class="btn btn-info btn-sm">Edit</button>
		</div>


	</div>


</div>

```

In the template file, functions and objects declared in the js file are referenced in the common
MVC Angular way. Special Bootstrap classes make much more attractive our UI. Angular helps us accomplish functionality which would require much more Js + CSS work. One example is the typeahead feature of Angular for the folder name:

```html
<label>Folder </label> <input type="text" ng-model="bookmark.folder"
	typeahead="folder.name for folder in folders | filter: $viewValue: startsWith" class="form-control"
	placeholder="Start typing for a folder">
```

when the user starts writing the name of the folder then the already created folder names starting from the typed characters appear in a dropdown list:

![Type ahead folder name][typeahead]

Another very nice thing is that there already a lot of ready to use Bootstrap components that with simple copy past can be cusomised easily. One such example is class badge with which you can inform the use how many bookmarks exist in one folder:


```Html
        <button type="button" ng-repeat="folder in folders" ng-click="getBookmarksInFolder(folder.name)" class="btn btn-default">
		{{ folder.name }} <span class="badge">{{folder.count}}</span>
        </button>
```

The result:

![Number of bookmarks][number-of-bookmarks]

## Conclusion
Building Web apps on the Java space has never been easier even for a backend developer. Spring Boot makes building RESTFUL APIs piece of cake and the same is done in the front end with frameworks like Angular and Bootstrap. As always of course someone needs to know the important stuff of all the underlying technologies like Java, Javascript, Html/CSS and Javascript. It is just that now the level of abstraction a developer is working with has been shifted upwards removing details and focusing on the important things.

###  

[bookmarks_list]: https://raw.githubusercontent.com/akalkatzakos/akalkatzakos.github.io/master/_images/20151120/bookmarks_list.png

[start_spring_io]: https://start.spring.io/ 

[post_create]:  https://raw.githubusercontent.com/akalkatzakos/akalkatzakos.github.io/master/_images/20151120/post_create.png 

[view_bookmark]:  https://raw.githubusercontent.com/akalkatzakos/akalkatzakos.github.io/master/_images/20151120/view_bookmark.png

[view_bookmarks]:  https://raw.githubusercontent.com/akalkatzakos/akalkatzakos.github.io/master/_images/20151120/view_bookmarks.png

[front-end-structure]: https://raw.githubusercontent.com/akalkatzakos/akalkatzakos.github.io/master/_images/20151120/front-end-structure.png

[typeahead]: https://raw.githubusercontent.com/akalkatzakos/akalkatzakos.github.io/master/_images/20151120/typeahead.png

[number-of-bookmarks]: https://raw.githubusercontent.com/akalkatzakos/akalkatzakos.github.io/master/_images/20151120/number-of-bookmarks.png
