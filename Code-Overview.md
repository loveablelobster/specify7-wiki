<h2>Introduction</h2>
<p>
This page is intended to present an overview of the front-end code for the Specify web application.
</p>
<p>
The Specify web app front-end is essentially a Javascript application which presents a user
interface for the Specify collection management system. The Javascript and various static resources
are served from a back-end server which also provides a REST api to the underlying database. This api
is directly utilized by the front-end to provide the user interface.
</p>

<h2>Third-Party Technology</h2>
<p>
A few Javascript libraries are utilized in the front-end. A basic understanding of these is necessary
to understand the Specify web app code.
</p>

<h3>JQuery</h3>
<p>
<a href="http://docs.jquery.com/Main_Page">JQuery</a> is heavily relied upon in this application. It
is used for DOM manipulation of the user visible web pages that constitute the UI. Additionally,
various resources (view definitions, the specify datamodel, schema localization, etc.) are
represented by XML exposed by the server. JQuery is used in querying these data structures.
Finally, heavy use is made of the <a href="http://api.jquery.com/category/deferred-object/">JQuery
deferred object</a> in order to allow concurrent fetching of data and rendering of UI. An
understanding of this <a href="http://en.wikipedia.org/wiki/Promise_(programming)">paradigm</a> is
essential for working with this code base.
</p>
<p>
The ajax requests to the back-end server are also brokered by JQuery.
</p>

<h3>RequireJS</h3>
<p>
The web app is of sufficient complexity that it is necessary to organize the code into separate
modules encapsulating logically separate concerns. Currently lacking a built-in module system,
Javascript benefits from the use of libraries to provide this functionality. I have
chosen <a href="http://requirejs.org/">RequireJS</a> for use in this project.
</p>
<p>
For a cursory understanding of the code base it is probably sufficient to understand how a module
definition is structured when using RequireJS. Most of the javascript files in this project take the
following form:
</p>

<pre>
define(['dependency1', 'dependency2', ...], function(localNameForDep1, localNameForDep2, ...) {
   ....
   module setup and local definitions
   ....
   return exported functionality
});
</pre>

<p>
For example,
</p>

<pre>
define(['jquery', 'mainview', 'specifyapi'], function($, MainView, api) {
   ...
});
</pre>

<p>
might, in a language with builtin modules, be the equivalent of:
</p>

<pre>
import jquery as $
import mainview as MainView
import specifyapi as api

...
</pre>

<p>
Given this structure, RequireJS ensures that all the necessary <code> .js </code> files are loaded
by the browser before the code is interpreted.
</p>

<h3>UnderscoreJS</h3>
<p>
<a href="http://documentcloud.github.com/underscore/">UnderscoreJS</a> is a utility library that
abstracts away many of the rough edges of Javascript in much the same way that JQuery abstracts away
complexities in the browser DOM interface. It also makes use of the familiar JQuery style whereby an
object is <em>wrapped</em> to provide extended functionality. Thus, if <code> foo </code> is an object,
then <code> _(foo) </code> is a wrapped object with special sauce.
</p>

<h3>Backbone.js</h3>
<p>
Resources that are fetched from the back-end must be represented in data structures in
Javascript. These structures then become associated with various UI elements in the
DOM. <a href="http://documentcloud.github.com/backbone/">Backbone</a> provides a standard set of
abstractions based around a MVC pattern that can be used to structure these data and UI
interactions. The main abstractions provided and their use in this application are as follows:
</p>
<ul>
  <li>
    <h4>Models</h4>
    <p>
      Models represent groups of related data that are generally synchronized with objects on the
      back end. In this project they are mostly used to represent instances of Specify datamodel
      objects. E.g. CollectionObjects, Accessions, Determinations, etc.
    </p>
    <p>
      The advantage of using Backbone here is that it provides out-of-the-box underpinnings for tasks like
      synchronizing with the server, parsing objects, observer pattern, etc.
    </p>
    <p>
      Sadly, the term <em>Model</em> is highly overloaded in this project and is used more-or-less
      interchangeably along with <em>Resource</em> to mean a variety of things. The situation for <em>View</em>
      and <em>Form</em> is similarly fraught.
    </p>
  </li>
  <li>
    <h4>Collections</h4>
    <p>
      Essentially these represent collections of Models. Predominately these are used
      to represent one-to-many collections, record sets, querycbx search results and the like.
    </p>
  </li>
  <li>
    <h4>Views</h4>
    <p>
      Views abstract the association of data with user interface. One thing to be aware of here is that I
      use the <code> View.render() </code> method somewhat differently than the Backbone documentation
      describes. Where the more standard semantics for that method imply that it may be called repeatedly
      to redraw the UI if the underlying model changes, I have adopted a more single-use style in which
      the method sets up the DOM for the particular UI and events are attached which modify it 'in-place'
      when changes occur.
    </p>
  </li>
</ul>

<h2>Architecture and Flow</h2>
<p>
The web app is essentially a system which takes a URI pointing to a resource and renders a form
based UI representing the data in that resource. Behaviors are attached to the rendered form that
permit the data to be modified and relationships to be navigated.
</p>
<img src="/static/img/specify_webapp_frontend.png">

<p>
The figure presents the architecture of the front-end in simplified schematic form. Rendering
proceeds mainly from top to bottom.
</p>

<ul>
  <li><p>
    Visiting a particular resource corresponds to accessing a particular
    URL. The <code>main.js</code> module examines the URL to determine the preliminary information
    needed to begin rendering the view. In this case, we will be producing a form for a single
    CollectionObject with primary key 12.
  </p></li>
  <li><p>
    Given the type of the object, <code>schema.js</code> returns a data structure representing the
    schema information for that type (i.e. fields, relationships, default forms, etc.). This data
    structure is sometimes given the name <code> specifyModel </code> in the code to differentiate
    it from what Backbone calls models, although the class itself is <code>schema.Model</code>.
  </p></li>
  <li><p>
    With the specifyModel and the object's id, there are a family of modules, <code>*api.js</code>,
    which conspire to retrieve the associated data from the back-end webservice. The retrieved data
    will eventually become available in the form of linked Backbone models and collections. The code
    base uses <em>resource</em> and <em>model</em> somewhat interchangeably to refer to these structures.
    </p>
    <h4>Aside regarding deferreds and <code> rget </code> </h4>
    <p>
    Because the resources are ultimately being retrieved asynchronously from the server, the api
    code in general returns deferred promises so that the UI can continue rendering and other
    resources can be fetched concurrently. The resources that are returned are specialized Backbone
    models which implement a <code>rget(fieldName)</code> method. This method extends the semantics
    of the regular models' <code> get </code> methods to return deferreds. This is necessary because
    a requested field may exist as a related object or even be a few steps down in a related object
    tree, e.g. <code>collector.agent.lastName</code>. If the related object(s) has not been previously
    fetched, we must request it from the server. This means a promise is the best that we can
    return.
    </p></li>
  <li><p>
    Once the schema info is available and the resource fetch is underway, control flows into
    the <code>populateform.js</code> module. First the schema info is passed to the <code>
    specifyform.js </code> module which determines the appropriate view for the object and parses
    the XML view definitions to produce a skeleton DOM which contains all the structure and
    information needed to produce the finished view. This process is independent of the specific
    data in the given resource and can happen concurrently with the fetch. This also means that the
    constructed DOMs could be serialized and replace the existing form definitions in future
    versions.
  </p></li>
  <li><p>
    The DOM is then supplemented with information from the schema localization tables. This step is
    also independent of the resource data fetch.
  </p></li>
  <li><p>
    Having obtained the DOM structure for the view, the populateform module will visit the various
    UI elements within. These elements carry attributes which determine what sort of UI they
    require, what fields they represent, whether they are read-only or required, etc. Populateform
    essentially dispatches on the UI type for each element to various other modules which are
    responsible for generating the corresponding UI. The figure sketches out the Subview, querycbx,
    picklist and uifield modules indicating how the first three may utilize the backend api to
    retrieve extra resources (pick list definitions, etc.) as necessary. The dashed red arrow
    illustrates how the Subview module makes recursive use of populateform. Not indicated in the
    figure are the various plugin modules that populateform may dispatch to via
    the <code>specifyplugins.js</code> module. All of the UI producing modules and plugins should be
    implemented as Backbone views in order to maintain a consistent interface and structure. A few
    of the existing modules need to be refactored to meet this goal.
  </p></li>
</ul>
