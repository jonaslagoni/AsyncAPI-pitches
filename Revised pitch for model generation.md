
# Summary

The following are the proposed pitch from @magicmatatjahu and me in regards to implementing the data modelling based the research done in #21. The specific research where done in #32, #33, #34 and #31 and the specific requirements found in #42.

We will not re-define the problem from #21 but rather focus on the solution elements. Also this is a revised pitch for the model generation 

## Solution Overview
* Implement an SDK which generates data models
* It must be possible to customize the data model generation
* It must be implement with the idea that multiple inputs should be supports

## Positive Side-Effects

If done right this will not only benefit the AsyncAPI community but anyone who wish to generate data models.

# Solution guidelines
The following are the solution guidelines and based on the requirements prioritized as `Must have` in #42. Follow up issues are required to resolve to a complete solution solving all the user stories. The non-functional requirements stated are however always expected to be followed. 


## Proposed solution
The following are the proposed solution which first outlines the different interactions that the library can have for then to dive into the generation process.

We propose to use React as our template language for the models and provide different out of box components for each language which can be used to fully customize the model. Below is an overview of the newly added modules, it is fairly high abstraction since implementation details are yet to be figured out. 

![Data model -Overview](https://user-images.githubusercontent.com/13396189/103298739-41884280-49fb-11eb-87e0-458c982c948e.png)

As seen on the diagram we introduce `Language Components`, these are described further down. `DataModeller` are the module which is the core of the library which handles the conversions of any input to its internal model representation in the `Input processor` and then rendering that component by using the existing `generator-react-sdk` render class and returning the model. What the diagram does not show is that we propose a simple output class which contain 0 to many generated models based on the input which might also contain other meta data such as name of model etc. The `CLI` module is for providing the user a way to generate raw models with fairly limited customization.


In the overview you can see different actors interacting with the library. The `User` interact directly through the CLI where `library, Nunjucks and React template` uses the model generation library. The `React template` also has the option of using the direct language components to build its own models from the ground up. The different interactions which should be possible with the model generation library are the following:
1. User wants to generate models through the CLI of the generator
1. Integrate the model generation in a library
1. Integrate the model generation in a React template
1. Integrate the model generation in a Nunjucks template
1. User wants to customize the generated models through the CLI of the generator
1. Customize the model generation in a library
1. Customize the model generation in a React template
1. Customize the model generation in a Nunjucks template

### The generation process
The generation process is split up into the following stages:
1. Input process
2. Rendering process

### The input process 

![Data model -Input](https://user-images.githubusercontent.com/13396189/103298753-4947e700-49fb-11eb-9a23-a5e428d1e0bb.png)

Proposed solution contains an input process stage before rendering to use a common format to pass to the models. This is because of the requirements for multiple inputs - `#E2`. This means it can handle AsyncAPI and JSON Schema inputs from the initial version as they are stated as must haves, without having to introduce dependency.

Although AsyncAPI message payload is a superset of JSON Schema draft 7 we most likely will handle that input a bit differently. Especially if we have to support input that have been through the parser beforehand. Therefore we introduce a `CommonInputModel` class which contains all the relevant information for a model to be rendered. This class can be seen as a wrapper to the underlying `CommonModel` since it might contain more then 1 model to render.

One part thats left out from the image and explanation is the input part regarding customization. This is something that we have not layed out how it should interact.

### The rendering process
We suggest using the already existing React SDK (`generator-react-sdk`)to render react components.

### Used in a library 
The following are examples to interactions with the library and how they could be done while also providing customization. Don't take notice in the specific customization examples on the actual arguments and configurations but the overall interaction. The arguments are to be finalized.

We have to be able to use the model library in any context, an example implementation could be the following:
```js
import { generateModel, Visibility} from '@asyncapi/model-generator/java'
function renderModel(schema){
    var modelContent = generateModels(
        schema,
        {
            package: 'com.asyncapi',
            dependencies: ['com.fasterxml.jackson.annotation.JsonFormat'],
            annotations: {
                properties: (property, propertyName) => {
                    if (property.format() !== 'date-time') return property
                    return annotate(property, 'JsonFormat', [{
                        shape: 'JsonFormat.Shape.STRING',
                        pattern: '"dd-MM-yyyy hh:mm:ss"',
                    }])
                },
                methods: (method) => {
                    const { name, visibility, isStatic, returnType, arguments } = method
                    annotate(method, 'SomeAnnotation', [{
                        someAnnotationKey: '"someAnnotationValue"',
                    }])
                }
            },
            methods: {
                hashCode: null, // Removes hashCode method from the generated result
                findEvenOdd: buildMethod({
                    name: 'findEvenOdd',
                    visibility: Visibility.PUBLIC,
                    static: true,
                    returnType: 'String',
                    parameters: [{
                        name: 'num',
                        type: 'int',
                        // default: 0,
                    }],
                    body: '// method implementation here...'
                }),
                // getters
                ...buildGetters(commonSchema),
                // setters
                ...buildSetters(commonSchema),
            },
        })
    ...
}
```

### Components to build
Based on the stated `must have` requirements and the investigation of correlation between languages the following are the proposed components which should be included in this issue. An important note, be careful when choosing between reducing duplicate code and reducing complexity, since one might introduce some common functionality to reduce duplicate code it will inevitably create complexity. Our suggestion is to reduce complexity in any turn possible over duplicate code, i.e. don't introduce common functionality between components of different languages.

These proposed components does not contain any specific props and we are leaving this up to the implementation phase to figure that out.

### Java
The following are components for Java which we see as must haves:
`<Package>`, `<Import>` (or `<Imports>`), `<Class>`, `<Constructor>`, `<Interface>`, `<Property>`, `<Method>`, `<Argument>` (as parameter in method/constructor), `<Enum>`, `<Annotation>`, `<Setter>` (or `<SetAccessor>`), `<Getter>` (or `<GetAccessor>`), `<Body>`. Getter and Setter could be include in `<Accessors>` component. String inside Method component could be treated as Body component.
​

Nice to have:
`<Lambda>`, [`<Record>`](http://tutorials.jenkov.com/java/record.html).


### TypeScript/JavaScript


The following are components for TypeScript/JavaScript which we see as must haves:
`<Module>` (for CommonJS), `<Import>` (or `<Imports>`), `<Class>`, `<Constructor>`, `<Method>`, `<Argument>` (as parameter in method/constructor), `<Setter>` (or `<SetAccessor>`), `<Getter>` (or `<GetAccessor>`), `<Function>`, `<Body>`, `<Property>`. Getter and Setter could be include in `<Accessors>` component. String inside Method component could be treated as Body component. `<Interface>`, `<Type>`, `<Decorator>`, `<Enum>`, `<Declare>`
​

Nice to have:
`<Variable>` (as standalone variable), then `<Object>`, `<Array>` etc..., `<Lambda>`.
​

### Python

The following are components for python which we see as must haves:
`<Import>` (or `<Imports>`), `<Class>`, `<Constructor>` (dunder `__init__` method), `<Property>`, `<Method>`, `<Argument>` (as parameter in method/constructor), `<Decorator>`, `<Setter>` (or `<SetAccessor>`), `<Getter>` (or `<GetAccessor>`), `<Function>`, `<Body>`. Getter and Setter could be include in `<Accessors>` component. String inside Method component could be treated as Body component. `<Enum>` (from `enum` package like `class Days(enum.Enum)`) - more info is [here](https://www.tutorialspoint.com/enum-in-python)

​
Nice to have:
`<Interface>` (this same as `<Class>` in Python case), `<DunderMethod>` (or maybe use normal `<Method>`?)


# Boundaries
### Only include
* Only focus on 1 language to start off with. If there is time another can be included.
* Start off with simple inputs such as AsyncAPI and JSON Schema.

### Please leave this alone
* For now the CLI should not be included in the solution.

### Watch out for 
* React might not be the best way to go, consider other options as well.
