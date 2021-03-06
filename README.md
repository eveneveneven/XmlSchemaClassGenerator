XmlSchemaClassGenerator
=======================

[![NuGet version](https://badge.fury.io/nu/XmlSchemaClassGenerator-beta.svg)](http://badge.fury.io/nu/XmlSchemaClassGenerator-beta)
[![Build status](https://ci.appveyor.com/api/projects/status/yhxiw0stmv5y7f6n/branch/master?svg=true)](https://ci.appveyor.com/project/mganss/xmlschemaclassgenerator/branch/master)
[![codecov.io](https://codecov.io/github/mganss/XmlSchemaClassGenerator/coverage.svg?branch=master)](https://codecov.io/github/mganss/XmlSchemaClassGenerator?branch=master)

A console program and library to generate 
[XmlSerializer](http://msdn.microsoft.com/en-us/library/system.xml.serialization.xmlserializer.aspx) compatible C# classes
from <a href="http://en.wikipedia.org/wiki/XML_Schema_(W3C)">XML Schema</a> files.

Features
--------

* Map XML namespaces to C# namespaces, either explicitly or through a (configurable) function
* Generate C# XML comments from schema annotations
* Generate [DataAnnotations](http://msdn.microsoft.com/en-us/library/system.componentmodel.dataannotations.aspx) attributes 
from schema restrictions
* Use [`Collection<T>`](http://msdn.microsoft.com/en-us/library/ms132397.aspx) properties 
(initialized in constructor and with private setter)
* Use either int, long, decimal, or string for xs:integer and derived types
* Automatic properties
* Pascal case for classes and properties
* Generate nullable adapter properties for optional elements and attributes without default values (see [below](#nullables))
* Optional support for PCL
* Optional support for [`INotifyPropertyChanged`](http://msdn.microsoft.com/en-us/library/system.componentmodel.inotifypropertychanged)
* Optional support for Entity Framework Code First (automatically generate key properties)
* Optionally generate interfaces for groups and attribute groups

Unsupported:

* Some restriction types
* Recursive choices and choices whose elements have minOccurs > 0 or nillable="true" (see [below](#choice))
* Possible name clashes and invalid identifiers when names contain non-alphanumeric characters
* Groups with maxOccurs > 0

Usage
-----

From the command line:

```
Usage: XmlSchemaClassGenerator.Console [OPTIONS]+ xsdFile...
Generate C# classes from XML Schema files.
Version 1.0.58.0
xsdFiles may contain globs, e.g. "content\{schema,xsd}\**\*.xsd", and URLs.
Append - to option to disable it, e.g. --interface-.

Options:
  -h, --help                 show this message and exit
  -n, --namespace=VALUE      map an XML namespace to a C# namespace
                               Separate XML namespace and C# namespace by '='.
                               One option must be given for each namespace to
                               be mapped.
                               A file name may be given by appending a pipe
                               sign (|) followed by a file name (like schema.
                               xsd) to the XML namespace.
                               If no mapping is found for an XML namespace, a
                               name is generated automatically (may fail).
  -o, --output=FOLDER        the FOLDER to write the resulting .cs files to
  -i, --integer=TYPE         map xs:integer and derived types to TYPE instead
                               of string
                               TYPE can be i[nt], l[ong], or d[ecimal].
  -e, --edb, --enable-data-binding
                             enable INotifyPropertyChanged data binding
  -r, --order                emit order for all class members stored as XML
                               element
  -c, --pcl                  PCL compatible output
  -p, --prefix=PREFIX        the PREFIX to prepend to auto-generated namespace
                               names
  -v, --verbose              print generated file names on stdout
  -0, --nullable             generate nullable adapter properties for optional
                               elements/attributes w/o default values
  -f, --ef                   generate Entity Framework Code First compatible
                               classes
  -t, --interface            generate interfaces for groups and attribute
                               groups (default is enabled)
  -a, --pascal               use Pascal case for class and property names (
                               default is enabled)
      --ct, --collectionType=VALUE
                             collection type to use (default is System.
                               Collections.ObjectModel.Collection`1)
      --cit, --collectionImplementationType=VALUE
                             the default collection type implementation to use (
                               default is null)
      --ctro, --codeTypeReferenceOptions=GlobalReference, GenericTypeParameter
                             the default CodeTypeReferenceOptions Flags to use (
                               default is unset; can be: GlobalReference,
                               GenericTypeParameter)
      --tvpn, --textValuePropertyName=VALUE
                             the name of the property that holds the text value
                               of an element (default is Value)
      --dst, --debuggerStepThrough
                             generate DebuggerStepThroughAttribute (default is
                               enabled)
```

From code:

```C#
var generator = new Generator
{
    OutputFolder = outputFolder,
    Log = s => Console.Out.WriteLine(s),
    GenerateNullables = true,
    NamespaceProvider = new Dictionary<NamespaceKey, string> 
    { 
        { new NamespaceKey("http://wadl.dev.java.net/2009/02"), "Wadl" } 
    }
    .ToNamespaceProvider(new GeneratorConfiguration { NamespacePrefix = "Wadl" }.NamespaceProvider.GenerateNamespace)
};

generator.Generate(files);
```

Specifying the `NamespaceProvider` is optional. If you don't provide one, C# namespaces will be generated automatically. The example above shows how to create a custom `NamespaceProvider` that has a dictionary for a number of specific namespaces as well as a generator function for XML namespaces that are not in the dictionary. In the example the generator function is the default function but with a custom namespace prefix. You can also use a custom generator function, e.g.

```C#
var generator = new Generator
{
    NamespaceProvider = new NamespaceProvider
    { 
        GenerateNamespace = key => ...
    }
};
```

Nullables<a name="nullables"></a>
---------------------------------

XmlSerializer has been present in the .NET Framework since version 1.1 
and has never been updated to provide support for nullables
which are a natural fit for the problem of signaling the absence or presence of a value type
but have only been present since .NET Framework 2.0.

Instead XmlSerializer has support for a pattern where you provide an additional bool property
with "Specified" appended to the name to signal if the original property should be serialized. 
For example:

```xml
<xs:attribute name="id" type="xs:int" use="optional">...</xs:attribute>
```

```C#
[System.Xml.Serialization.XmlAttributeAttribute("id", Form=System.Xml.Schema.XmlSchemaForm.Unqualified, DataType="int")]
public int Id { get; set; }

[System.Xml.Serialization.XmlIgnoreAttribute()]
public bool IdSpecified { get; set; }
```

XmlSchemaClassGenerator can optionally generate an additional nullable property that works as an adapter to both properties:

```C#
[System.Xml.Serialization.XmlAttributeAttribute("id", Form=System.Xml.Schema.XmlSchemaForm.Unqualified, DataType="int")]
public int IdValue { get; set; }
        
[System.Xml.Serialization.XmlIgnoreAttribute()]
public bool IdValueSpecified { get; set; }

[System.Xml.Serialization.XmlIgnoreAttribute()]
public System.Nullable<int> Id
{
    get
    {
        if (this.IdValueSpecified)
        {
            return this.IdValue;
        }
        else
        {
            return null;
        }
    }
    set
    {
        this.IdValue = value.GetValueOrDefault();
        this.IdValueSpecified = value.HasValue;
    }
}
```

Choice Elements<a name="choice"></a>
------------------------------------

The support for choice elements differs from that [provided by xsd.exe](http://msdn.microsoft.com/en-us/library/sa6z5baz).
Xsd.exe generates a property called `Item` of type `object` and, if not all choices have a distinct type, 
another enum property that selects the chosen element.
Besides being non-typesafe and non-intuitive, this approach breaks apart if the choices have a more complicated structure (e.g. sequences),
resulting in possibly schema-invalid XML.

XmlSchemaClassGenerator currently simply pretends choices are sequences.
This means you'll have to take care only to set a schema-valid combination of these properties to non-null values.

Interfaces<a name="interfaces"></a>
-----------------------------------

Groups and attribute groups in XML Schema are reusable components that can be included in multiple type definitions. XmlSchemaClassGenerator can optionally generate interfaces from these groups to make it easier to access common properties on otherwise unrelated classes. So

```XML
<xs:attributeGroup name="Common">
  <xs:attribute name="name" type="xs:string"></xs:attribute>
</xs:attributeGroup>

<xs:complexType name="A">
  <xs:attributeGroup ref="Common"/>
</xs:complexType>

<xs:complexType name="B">
  <xs:attributeGroup ref="Common"/>
</xs:complexType>
```

becomes

```C#
public partial interface ICommon
{
  string Name { get; set; }
}

public partial class A: ICommon
{
  public string Name { get; set; }
}

public partial class B: ICommon
{
  public string Name { get; set; }
}
```

Contributing
------------

Pull requests are welcome :)
