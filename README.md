jaxb2-rich-contract-plugin
==========================

Current Version: 1.1.5

This module is a collection of several plugins for the JAXB2 (Java API for XML binding) "XSD to Java Compiler" (XJC).
These plugins are intended to add support for additional contracts to the classes generated by XJC.
Currently, there are 5 plugin classes:

1. **group-interface**: When using `<attribute-group>` or `<group>` elements in an XSD, they are transformed as interface definitions, and any complexTypes using the groups will be generated as classes implementing this interface.
2. **constrained-properties**: Will generate a complexTypes element members as bound and/or constrained properties as per the JavaBeans spec.
3. **clone**: Will generate a simple deep "clone" method for the generated classes based on the heuristic that it only makes sense to traverse further down in the cloned object tree for members of types that are actually cloenable themselves.
	Also can generate a "partial clone" method, that takes a `PropertyPath` object which represents an include/exclude rule for nodes in the object tree to clone. Excluded nodes will not be cloned and left alone.
	Optionally, corresponding copy constructors can also be generated.
4. **immutable**: Will make generated classes immutable. Only makes sense together with "fluent-builder" plugin (see below), or any other builder or initialisation facility, like the well-known "value-constructor" plugin.
5. **fluent-builder**: Generates a builder class for every class generated. Builders are implemented as inner classes,
	static methods are provided for a fluent builder pattern in the form `MyClass.builder().withPropertyA(...).withPropertyB(...).build()`.
	This is particularly useful together with `-Ximmutable` (see above)


### Usage
#### General
jaxb2-rich-contract-plugin is a plugin to the XJC "XML to Java compiler" shipped with the
reference implementation of JAXB, included in all JDKs since 1.6.
It is targeted on version 2.2 of the JAXB API.
In order to make it work, you need to:

* Add the jar file to the classpath of XJC

* Add the JAXB 2.2 XJC API to the classpath of XJC, if your environment is running by default under JAXB 2.1 or lower.

* Add the corresponding activating command-line option to XJC's invocation,
  see below for details of each of the plugins

#### From Maven
There is a maven repository for this project under:

http://maven.klemm-scs.com/release

Add this repository to your pom.xml:

	<pluginRepositories>
		<pluginRepository>
			<releases>
		        <enabled>false</enabled>
		        <updatePolicy>always</updatePolicy>
		        <checksumPolicy>warn</checksumPolicy>
		    </releases>
			<id>jaxb2-plugins</id>
			<name>JAXB2 XJC Plugin Repository</name>
			<url>http://maven.klemm-scs.com/release</url>
			<layout>default</layout>
		</pluginRepository>
	</pluginRepositories>
	


You should add "maven-jaxb21-plugin" or "maven-jaxb22-plugin" to your `<build>`
configuration.
Then add "jaxb2-rich-contract-plugin" as an XJC plugin ("plugin for plugin") to the maven plugin declaration:

    <build>
        <plugins>
            <plugin>
                <groupId>org.jvnet.jaxb2.maven2</groupId>
                <artifactId>maven-jaxb21-plugin</artifactId>
                <version>0.8.3</version>
                <executions>
                    <execution>
                        <id>xsd-generate-2.1</id>
                        <phase>generate-sources</phase>
                        <goals>
                            <goal>generate</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <schemaIncludes>
                        <schemaInclude>**/*.xsd</schemaInclude>
                    </schemaIncludes>
                    <strict>true</strict>
                    <verbose>true</verbose>
                    <extension>true</extension>
                    <removeOldOutput>true</removeOldOutput>
                    <args>
                        <arg>-Xgroup-contract</arg>
                        <arg>-Xconstrained-properties</arg>
                        <arg>-Xclone</arg>
                        <arg>-Ximmutable</arg>
                        <arg>-Xfluent-builder</arg>
                        <arg>...</arg>
                    </args>
                    <plugins>
                        <plugin>
                            <groupId>com.kscs.util</groupId>
                            <artifactId>jaxb2-rich-contract-plugin</artifactId>
                            <version>1.1.5</version>
                        </plugin>
                    </plugins>
                    <dependencies>
                        <!-- Put this in if your default JAXB version is 2.1 or lower,
                        or if &quot;tools.jar&quot; isn't in your classpath -->
       			<dependency>
       				<groupId>com.sun.xml.bind</groupId>
       				<artifactId>jaxb-xjc</artifactId>
       				<version>2.2.7</version>
       			</dependency>
       			<dependency>
       				<groupId>com.sun.xml.bind</groupId>
       				<artifactId>jaxb-impl</artifactId>
       				<version>2.2.7</version>
       			</dependency>
       		    </dependencies>
                </configuration>
            </plugin>
        </plugins>
    </build>

Note: the `<extension>` flag must be set to "true" in order to make XJC accept any extensions at all.

Note: jaxb2-rich-contract-plugin implements JAXB and XJC APIs version 2.2.7. You most likely will have
	to add the dependencies to these libraries to your classpath effective at XJC runtime. See the
	`dependencies` element above on how to do this.


### Version History
* **1.0.0**:	Initial Version
* **1.0.1**:	Added constrained-property plugin
* **1.0.2**:	Added partial clone method generation
* **1.0.3**:	Improvements in partial clone
* **1.0.4**:	Added fluent builder and immutable plugins
* **1.0.5**:    Added chainable fluent builder support
* **1.1.0**:    New: Ximmutable, Copy constructor support,
				fluent-builder copy from instance support,
				general fixes.
				Removed option to generate fluent builders without
				chained builder support.
* **1.1.1**:	New: Type-safe selector support for partial clone/copy logic.
* **1.1.2**:	Big fixes in selector logic
* **1.1.3**:	Minor bug fixes in fluent-builder
* **1.1.4**:	Fixed an error in fluent-builder where an initialization method wasn't properly overridden in derived builder classes, leading to the wrong builder type being returned when using chained sub-builders.
* **1.1.5**:    Fixed error in Release Build process

group-interface
--------------------

### Motivation
Out of the box, the only polymorphism supported by classes generated from an XSD is the `<extension>` notion,
transformed directly into an inheritance relationship by XJC.
However, pure inheritance relationships are often inflexible and do not always reflect the intention
of generating a "contract" that implementing classes must follow.

With this plugin, it is possible for your application code to treat classes using one or more specific model or attribute groups in a comman way. Objects can be initialised or used via the interface.

### Function
For definition of contracts, two additional XSD constructs, the `<group>` and `<attributeGroup>`,
are readily available in XSD, but they're currently ignored by standard XJC and simply treated as an inclusion of
elements or attributes into a generated class definition.
The group-interface plugin changes that and generates an `interface` definition for each `group` or `attributeGroup`
found in your model, defines the attributes or elements declared in the groups as get and set methods on the interface,
and makes each generated class using the group or attributeGroup implement this interface.

### Usage
See below on how to add the jaxb2-rich-contract-plugin to your plugin path when building your project with maven.
group-interface is activated by adding the `-Xgroup-contract` command-line option to your XJC invocation.
For group-interface, there are currently no further command line options.

### Limitations
* Currently, interface definitions are only generated if there actually is a
  `complexType` using the respective `group` or `attributeGroup`. This is because, for
  the sake of simplicity, the plugin copies back the method definitions in the
  interfaces from the implementing classes, rather than generating the method
  definitions entirely on its own, taking into account possible other plugins etc.

* There should be an option to limit the contract only to property getter methods, or
  to extend it to the fluent-interface "withXXX"-Methods, when the "fluent-interface"
  plugin is active.

### Bugs
* Currently none known.



constrained-properties
----------------------

### Motivation
Many GUI applications use data binding to connect the data model to the view components. The JavaBeans standard
defines a simple component model that also supports properties which send notifications whenever the are about to be changed,
and there are even vetoable changes that allow a change listener to inhibit modification of a property.
While the JAvaBeans standard is a bit dated, data binding and property change notification can come in handy in many situations,
even for debugging or reverse-engineering existing code, because you can track any change made to the model instance.

### Function
constrained-properties generates additional code in the property setter methods of the POJOs generated by XJC that
allow `PropertyChangeListener`s and `VetoableChangeListener`s to be attached to any instance of a XJC-generated class.

Currently, **indexed properties** are NOT supported in the way specified by JavaBeans, but instead, if a property represents a collection,
a collection proxy class is generated that supports its own set of collection-specific change notifications, vetoable and other.
This decision has been made because by default XJC generates collection properties rather than indexed properties,
and indexed properties as mandated by JavaBeans are generally considered "out of style".

### Usage
Activate the plugin by giving command-line option `-Xconstrained-properties` to XJC.

Other options supported:

#### `-setter-throws=`y/n
Generate the setter method with "throws PropertyVetoException" if constrained properties are
used. If no, only a RuntimeException is thrown on a PropertyVeto event. Default: n
							
#### `-constrained=`y/n
Generate constrained properties, where a listener can inhibit the property change. Default: y

#### `-bound=`y/n
Generate bound properties. Default: y

#### `-generate-tools=`y/n
To support Collection-specific change events and behavior, additional classes are required.
If you set this option to "yes", these auxiliary classes will be generated into the source
code along with the generated JAXB classes. If you set this to "no", you will have to include
the plugin artifact into the runtime classpath of your application.

### Limitations
* The JavaBeans standard is only loosely implemented in the generated classes.

* Indexed Properties as defined in JavaBeans are not supported.

* The CollectionChange behavior implemented by the classes is not yet documented
  and non-standard.

### Bugs
* Currently none known.

clone
-----

### Motivation
Sometimes it is necessary to create a deep copy of an object. There are various approaches to this. The clone plugin
tries to do it in a simple and reliable way.

### Function
The `clone` plugin generates a deep clone method for each of the generated classes, based on the following assumptions:
* Instances of any other class generated from the same XSD model are cloneable by the same semantics as "this".

* Objects implementing `java.lang.Cloneable` and not throwing "CloneNotSupportedException" are also reliably cloneable
  by their "clone" Method.

* Objects not implementing `java.lang.Cloneable` or primitive types are assumed to be immutable,
  their references are copied over, they are not cloned.

* Optionally, generates a "partial clone" method that takes a `PropertyPath` instance which represents a
  specification of the nodes in the object tree to clone. The PropertyPath is built up by an intuitive builder
  pattern:

`final PropertyPath excludeEmployees = PropertyPath.includeAll().include("company").exclude("employees").build();`

* As of release 1.1.1, there is also a type-safe way to build a PropertyPath instance by using a generated classes' `Selector` sub structure. The following will generate the same selection as above:

`final PropertyPath excludeEmployees = Business.Select.root(true).company(true).employees(false).build()`



Then, you would partially clone an object tree like this:

    final BusinessPartner businessPartnerClone = businessPartner.clone(excludeEmployees);


### Usage
Plugin activation: `-Xclone`.
Options:
#### `-copy-constructor=`y/n
Generates a copy constructor.

#### `-narrow=`y/n
If copy constructors are created, they are generated in such a way that, in order to copy descendant objects,
they call the copy constructors of any descendant property types found in the tree. This will result in an object
tree that contains only the "narrowest" possible representation. For properties declared with an
abstract type, their `clone()`-Methods will be called instead, resulting in an exact runtime copy. Default: no.

#### `-clone-throws=`y/n
Declare "clone()"-Method to throw "CloneNotSupportedException" any of the cloneable child
objects have a "clone" method that declares `CloneNotSupportedException` being thrown.
The JDK spec says objects should declare their "clone" method with "throws CloneNotSuppoertedException"
in order to enable subclasses to inhibit cloning even if their superclass declares "Cloneable".
In pratice, however, this doesn't make much sense and is against object-oriented principles.
All classes implementing `Cloneable` should really be cloneable and NOT throw
a `CloneNotSupportedException`.
Ordinary Exceptions during cloning are rethrown as `RuntimeExceptions`,
if this is set to "no", this is also true for any exceptions thrown by descendant objects,
no matter whether they are `CloneNotSupported`or anything else. If this is "yes", any possible
`CloneNotSupportedException` will be declared in the throws clause. This option will not
affect generated "partial" clone methods. Default: no.

#### `-partial-clone=`y/n
Create partial clone method and - if constructor generation is selected, a partial copy constructor. (see above)

#### `-generate-tools=`y/n
Generate prerequisite classes like e.g. `PropertyPath` as source files into the generated source
packages. If you say 'no' here, you will have to add the jaxb2-rich-contract-plugin jar to your
runtime classpath.

### Limitations
* The `-narrow` option is a somewhat special use case and should be used carefully.


immutable
---------

### Motivation
Contemporary programming styles include making objects immutable as much as possible, to minimise
side effects and allow for functional programming patterns.

### Function
This plugin simply makes all "setXXX" methods "protected", thus preventing API consumers to modify
state of instances of generated classes after they have been created. This only makes sense together with
another plugin that allows for initialization of the instances, like e.g. the included `fluent-builder` plugin.
For collection-valued properties, `-Ximmutable` wraps all collections in a `Collections.unmodifiableCollection`,
so collections are also made immutable.

### Usage
Plugin activation: `-Ximmutable`

### Limitations
* Access level "protected" may not be strict enough to prevent state changes.
* If you activate plugins like "fluent-api" or the like, these plugins may circumvent the protection provided by the `immutable` plugin.


fluent-builder
--------------

### Motivation
There already is the widely used "fluent-api" plugin for XJC.
That, however isn't a real builder pattern since there is no strict programmatic distinction between initialization
and state change in fluent-api.

fluent-builder now creates a real "Builder" pattern, implemented as an inner class to the generated classes.

### Function
fluent-builder creates a static inner class for every generated class representing the builder, and a static
method on the generated class to create a builder.

If the "immutable" plugin is also activated, publicly exposed collections will be immutable, too.

Example use in code:

	MyElement newElement = MyElement.builder().withPropertyA(...).withPropertyB(...).addCollectionPropertyA(...).build();

In addition, new instances can be created as copies of existing instances using the builder, with an optional modification by othe builder methods:

	MyElement newElement = MyElement.copyOf(oldElement).withPropertyA(...).withPropertyB(...).build();
	
The "partial" copy introduced in the "clone" plugin will work here as well:

	PropertyPath selection = MyElement.Select.root(true).propertyA(true).propertyAB(false).build();
	MyElement newElement = MyElement.copyOf(oldElement, selection).withPropertyA(...).withPropertyB(...).build();
	
Often, properties of generated classes represent containment or references to generated classes in the same model.
The fluent-builder plugin lets you initialise properties of such a type - if it isn't an abstract type - by using sub-builders ("chained" builders) in the following way, given that both A and B are types defined in the XSD model, and A has a property of type B, and B has three properties of type String, x,y, and z:

	A newA = A.builder().withB().withX("x").withY("y").withZ("z").end().build();
	
Of course, this plugin is most useful if `immutable` is also activated.

### Usage
Plugin activation: `-Xfluent-builder`

Options:
#### `-narrow=`y/n
When creating a builder via `copyOf()`, the narrowest possible copy of the input object is instantiated.
See also the documentation on `-Xclone` for a discussion.

#### `-partial-clone=`y/n
If set to true, generates a `copyOf()`-Method that takes an additional `PropertyPath` parameter to limit
the copying to a certain depth or certain nodes in the object tree.


### Limitations

* It generates a large amount of code.

* Note: Shared builder instances are NOT thread-safe by themselves.



