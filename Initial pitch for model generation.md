The following are the proposed pitch from @magicmatatjahu and me in regards to implementing the data modelling based the research done in #21. The specific research where done in #32, #33, #34 and #31 and the specific requirements found in #42.


# Solution guidelines
The following are the solution guidelines and based on the requirements prioritized as `Must have` in #42. Follow up issues are required to resolve to a complete solution solving all the user stories. The non-functional requirements stated are however always expected to be followed. 

# Proposed solution
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

## The generation process
The generation process is split up into the following stages:
1. Input process
2. Rendering process

### The input process 

![Data model -Input](https://user-images.githubusercontent.com/13396189/103298753-4947e700-49fb-11eb-9a23-a5e428d1e0bb.png)

Proposed solution contains an input process stage before rendering to use a common format to pass to the models. This is because of the requirements for multiple inputs based on requirement `#E2`. This means it can handle AsyncAPI and JSON Schema inputs from the initial version as they are stated as must haves, without having to introduce dependency.

Although AsyncAPI message payload is a superset of JSON Schema draft 7 we most likely will handle that input a bit differently. Especially if we have to support input that have been through the parser beforehand. Therefore we introduce a `CommonInputModel` class which contains all the relevant information for a model to be rendered. This class can be seen as a wrapper to the underlying `CommonModel` since it might contain more then 1 model to render.

### The rendering process
Already implemented in the `generator-react-sdk`, see the specific details there.

## Interactions
The following are examples to interactions with the library and how they could be done while also providing customization. Don't take notice in the specific customization examples on the actual arguments and configurations but the overall interaction. The arguments are to be finalized.

### Used in a library 
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
    return class;
}
```

### Used in a Nunjucks template
With the Nunjucks templates the model generation can be generated using the following hook setup:
```js
const JaveModelGeneration = require('@asyncapi/model-generator/java');
const fs = require('fs');
const Path = require('path');
module.exports = {
	'generate:after': async (generator) => {
		const allMessages = generator.asyncapi.allMessages();
                for (const [messageId, message] of allMessages) {
                    const payloadSchema = message.payload();
                    var {models} = JaveModelGeneration.generateModel(
                        payloadSchema, 
                        ...
                        )
                    const targetDir = Path.join(options.generatorTargetDir, options.targetDir);
                    await fs.promises
                        .mkdir(targetDir, { recursive: true })
                        .catch(console.error);
                    fs.mkdirSync(targetDir, { recursive: true });
                    fs.writeFileSync(
                        Path.join(
                        targetDir,
                        `${pascalCase(schemaName)}.${options.fileExtension}`
                        ),
                        model[0]
                    );
                }
	}
};
```
It would be possible to provide the AsyncAPI object directly where all payload schemas would be generated instead of iterating messages manually. 


### Used in a React template
Same as for Nunjucks using hooks.


We could do something as the following as well, but it seems silly to have two wrappers (`Model` component and `generateModel` method) and in the end be confusing since it adds nothing extra.
<details>
  <summary>See example</summary>
  
```js
import { Model, Visibility} from '@asyncapi/generator-react-sdk/java'
function renderSchema(schema) {
    return 
    <Model
        schema={schema}
        annotateProperties={
            (prop) => {
                if (prop.format() !== 'date-time') return prop
                return annotate(
                    prop, 
                    'JsonFormat', 
                    [{
                        shape: 'JsonFormat.Shape.STRING',
                        pattern: 'dd-MM-yyyy hh:mm:ss',
                    }]
                )
            }
        }
        annotateMethods={method => {
            const { name, visibility, isStatic, returnType, arguments } = method
            annotate(method, 'SomeAnnotation', [{
                someAnnotationKey: '"someAnnotationValue"',
            }])
        }}
        methods={{
            hashCode: null, // Removes hashCode method from the generated result
            findEvenOdd: {
                visibility: Visibility.PUBLIC,
                static: true,
                returnType: 'String',
                parameters: [{
                    name: 'num',
                    type: 'int',
                    // default: 0,
                }],
                body: '// method implementation here...'
            }
        }}
        buildGetters={true}
        buildSetters={true}
    />   
}
```

</details>


### Writing your own model from scratch

C# example model using the language react components, the idea is for all the languages to have their own components for all the valid concepts i.e. constructor, class, arguments etc.

```js
import { Namespace, Class, Property, Set, Get, Metadata, Constructor, Body, Argument, Method, Body} from '@asyncapi/generator-react-sdk/csharp'
function renderSchema(commonModel) {
    return <File name={commonModel.uid()}>
        <Namespace name="temp">
            <Class name={commonModel.uid()}>
                {
                    commonModel.properties().map((propertyName, property) => {
                        return <Property 
                            property={property} 
                            private 
                            name={propertyName}>
                            <Metadata>
                            [Serializable]
                            </Metadata>
                            <Set public />
                            <Get public />
                        </Property>
                    })
                }
                //Custom constructor beside default constructor
                <Constructor>
                    <Argument type="string" name="displayName" />
                    <Argument type="string" name="email" />
                    <Body>
                        displayName = displayName;
                        email = email;
                    </Body>
                </Constructor>

                //Add some custom method
                <Method public name="someMethod">
                    <Argument type="string" name="displayName" />
                    <Argument type="string" name="email" />
                    <Body>
                        return email == displayName;
                    </Body>
                </Method>
            </Class>
        </Namespace>
    </File>
}
```

There are examples, which show how it could be look in different languages: https://github.com/asyncapi/shape-up-process/issues/31#issuecomment-751710359

One way this render function can be used with the model generator is to provide the file location of the model, notice the `modelTemplateFile` variable. The renderer should be able to transpile that file and use it to generate the models.
```js
const JaveModelGeneration = require('@asyncapi/model-generator/java');
const fs = require('fs');
const Path = require('path');
module.exports = {
	'generate:after': async (generator) => {
                const modelTemplateFile = "file://./templatefile.js"
		const allMessages = generator.asyncapi.allMessages();
                for (const [messageId, message] of allMessages) {
                    const payloadSchema = message.payload();
                    var {models} = JaveModelGeneration.generateModel(
                        payloadSchema, 
                        modelTemplateFile
                        ...
                        )
                    const targetDir = Path.join(options.generatorTargetDir, options.targetDir);
                    await fs.promises
                        .mkdir(targetDir, { recursive: true })
                        .catch(console.error);
                    fs.mkdirSync(targetDir, { recursive: true });
                    fs.writeFileSync(
                        Path.join(
                        targetDir,
                        `${pascalCase(schemaName)}.${options.fileExtension}`
                        ),
                        model[0]
                    );
                }
	}
};
```

## CLI
Some people might only want to get the model generation trough a CLI such as what the generator provides. CLI of course have the inherit disadvantage of providing limited customization to the model generation process. This is because of the limitation in what you can provide as arguments. 

We propose creating a custom CLI for the library we however suggest looking into how easy it is to wrap different CLI's within one another such as if we in the future decide to add a general `asyncapi` CLI we would be able to reuse the implementation in some way as well as a standalone.

## Components to build
Based on the stated `must have` requirements and the investigation of correlation between languages the following are the proposed components which should be included in this issue. An important note, be careful when choosing between reducing duplicate code and reducing complexity, since one might introduce some common functionality to reduce duplicate code it will inevitably create complexity. Our suggestion is to reduce complexity in any turn possible over duplicate code, i.e. don't introduce common functionality between components of different languages.

These proposed components does not contain any specific props and we are leaving this up to the implementation phase to figure that out.

### Java
The following are components for Java which we see as must haves:
`<Package>`, `<Import>` (or `<Imports>`), `<Class>`, `<Constructor>`, `<Interface>`, `<Property>`, `<Method>`, `<Argument>` (as parameter in method/constructor), `<Enum>`, `<Annotation>`, `<Setter>` (or `<SetAccessor>`), `<Getter>` (or `<GetAccessor>`), `<Body>`. Getter and Setter could be include in `<Accessors>` component. String inside Method component could be treated as Body component.
​
Nice to have:
`<Lambda>`, [`<Record>`](http://tutorials.jenkov.com/java/record.html).
​
### TypeScript/JavaScript
​
The following are components for TypeScript/JavaScript which we see as must haves:
`<Module>` (for CommonJS), `<Import>` (or `<Imports>`), `<Class>`, `<Constructor>`, `<Method>`, `<Argument>` (as parameter in method/constructor), `<Setter>` (or `<SetAccessor>`), `<Getter>` (or `<GetAccessor>`), `<Function>`, `<Body>`, `<Property>`. Getter and Setter could be include in `<Accessors>` component. String inside Method component could be treated as Body component. `<Interface>`, `<Type>`, `<Decorator>`, `<Enum>`, `<Declare>`
​
Nice to have:
`<Variable>` (as standalone variable), then `<Object>`, `<Array>` etc..., `<Lambda>`.
​
### Python
The following are components for python which we see as must haves:
`<Import>` (or `<Imports>`), `<Class>`, `<Constructor>` (dunder `__init__` method), `<Property>`, `<Method>`, `<Argument>` (as parameter in method/constructor), `<Decorator>`, `<Setter>` (or `<SetAccessor>`), `<Getter>` (or `<GetAccessor>`), `<Function>`, `<Body>`. Getter and Setter could be include in `<Accessors>` component. String inside Method component could be treated as Body component. `<Enum>` (from `enum` package like `class Days(enum.Enum)`) - more info is [here](https://www.tutorialspoint.com/enum-in-python),
​
Nice to have:
`<Interface>` (this same as `<Class>` in Python case), `<DunderMethod>` (or maybe use normal `<Method>`?)

# Discussions and solutions
The following are the discussions and answers to questions we have come across while forming this proposed solution:
1. What is the limitations of the CLI customization
1. How to integrate the solution with a template while making it customizable
1. How to create an entire new model from scratch in a non-react template setting
1. How should the library be interacted with

## Answer to discussion number 1
The following are the answers which we have discussed and decided upon:
1. The generator approach, by creating a template wrapper around the library we can provide the model generation through the already existing generator CLI. It however introduce a problem with only AsyncAPI files can be provided and not for example a JSON Schema file on its own. 
1. Custom CLI wrapper for the library.

**We agreed to go with the custom CLI instead, however we have to keep in mind that that in the future we might want to wrap our CLI's with an `asyncapi` CLI and should have extensibility in mind when implementing it.**

## Answer to discussion number 2
The following are the answers which we have discussed and decided upon:
1. Create a template configuration such as `datamodel` whose value should reflect what model language are desired i.e. `"datamodel" : "java"`.
1. Use hooks to interact with the library directly.

**We agreed to go with the second answer and use hooks directly, if we later find a smarter way to do things that should easily be added to the generator.**

## Answer to discussion number 3
The following are the answers which we have discussed and decided upon:
1. It should be possible to implement a wrapper for Nunjucks if desired.
2. Custom transpilation process to use jsx.

## Answer to discussion number 4
We had some discussion in regards to how to interact with the library and these are those discussions
### Common configuration
When using the library approach should it generalize the usage such as:
```js
import { generateModelFor, languages} from '@asyncapi/model-generator'
var string = generateModelFor(
    ('java' || languages.JAVA), 
    schema, 
    ...
```
or be specific for each language such as:
```js
import { generateModel } from '@asyncapi/model-generator/java'
var string = generateModel(
    schema, 
    ...
```

**We agreed on using the second answer and instead of having one method where everything goes through, you select the language based on the import.**

### Return type of the library interaction
Should the return type of the library interaction be on the form (notice the `s` in the function call):
```js
var {modelContent, filename, filepermissions} = JaveModelGeneration.generateModel(...)
var {models} = JaveModelGeneration.generateModels(...)
var model = models[0]
```
or just always contain the following:
```js
var {models} = JaveModelGeneration.generateModels(...)
var model = models[0]
```

The last approach is more dynamic and offers only one way to interact with the library, and it does not require you to have any information about how many models are to be returned.

**We agreed on the second approach since it is less confusing for people and the output is always the same.**
