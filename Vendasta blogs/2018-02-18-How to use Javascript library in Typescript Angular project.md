<h1>How to use Javascript library in Typescript Angular project</h1>

*Originally posted at https://vendasta-blog.appspot.com/blog/BL-VJKFZ2J4/*

In the recent Hackathon, I spent my time on introducing a Gantt chart to our team dashboard. The codebase of the team dashboard is an Angular 2 project written in Typescript. I tried several Gantt packages, the one I liked most is a Javascript package <a href="https://dhtmlx.com/docs/products/dhtmlxGantt/" target="_blank">dhtmlxGantt</a>. But how could we use this Javascript package in the project?

<h4>Step 1. Find the type definition of Javascript library</h4>
Typescript is the strict syntactical superset of JavaScript. The functions and variables can have defined types in Typescript, while it is not important in Javascript. When we are going to use an external JS library we need to declare type definitions for TypeScript. Do we need to write it? Before writing it ourselves, there are several places we should check out.

<h5>Inside the JS library</h5>
Some JS libraries, like <a href="https://github.com/thlorenz/brace" target="_blank">brace</a>, provide the type definition files(.d.ts) which define types and standardize the arguments for TS. In this case, we can use the library directly. 
Unfortunately, dhtmlxGantt does not provide .d.ts file in the library.

<h5>DefinitelyTyped</h5>
<a href="https://github.com/DefinitelyTyped/DefinitelyTyped" target="_blank">DefinitelyTyped</a> is a great repository holding the type definitions of thousands JS libraries. We can use <a href="https://microsoft.github.io/TypeSearch/" target="_blank">Type Search</a> to find the declaration file for the libraries we need, and I found `@types/dhtmlxgantt` here.

<h5>npm and Github</h5>
Before we move on, I would like to suggest to search on npm and Github for personal packages/repositories that related to the JS library you want to use. Some of them could even be a better choice than the package in DefinitelyTyped. 
For example, <a href="https://github.com/dking3876/gantt-ui-component" target="_blank">gantt-ui-component</a> is also a wonderful TS wrapper of dhtmlxGantt.

If we failed to find the type definition, we have to write it ourselves. But it is another story, we can discuss it someday later.

<h4>Step 2. Install the library in the project</h4>
In order to use the library, we need to install the library in the project. For example, in the case of dhtmlxGantt, we should run following commands:
```
npm install dhtmlx-gantt --save
npm install @types/dhtmlxgantt --save-dev
```

These two packages will appear in the "dependencies" and "devDependencies" section of the "package.json" file.

<h4>Step 3. Import the library into the component</h4>
Now, we can import the library into the component file and use the library.

```typescript
import 'dhtmlx-gantt';

// or more explicitly,
// import * as gantt from 'dhtmlx-gantt';

@Component({
    selector: "gantt-chart",
    template: "<div #gantt></div>",
    styleUrls: ['./gantt-chart.component.scss']
})
export class GanttChartComponent implements OnInit {
    @ViewChild("gantt") ganttContainer: ElementRef;

    ngOnInit(){
        gantt.init(this.ganttContainer.nativeElement);
    }
}
```


<h4>Step 4. Import the stylesheet</h4>
At this point, the Gantt chart should be present on the page, well, super ugly. This is because the styling is missing, so we want to import the style sheet file(.css) in dhtmlxgantt library into our project.

One approach to import an external css file from node_modules is importing the css file in the `gantt-chart.component.scss` file and applying it with Shadow Piercing combinators.

```css
::ng-deep { // shadow piercing
    @import "../../node_modules/dhtmlx-gantt/codebase/dhtmlxgantt.css";
}

:host { // styles in the element that hosts the component
    display: block;
    position: relative;
    width: 100%;
}
```

The reason we need shadow piercing combinators is that component styles apply only to the HTML in the component's own template, while we want to apply them to the child component views. A tricky thing is that `::ng-deep`, as well as its alias `/deep/` and `>>>` is being deprecated. But as long as there is no clear roadmap of replacing them, it should be safe to keep on using the `::ng-deep` combinator.

Some CLI tools might not support such relative path of the css file. I ran into this problem in our team dashboard project with webpack 3 as the CLI tool, so I defined an alias of the css file in webpack.config.js.

```
config.resolve = {
  extensions: ['.ts', '.js', '.scss', '.html'],
  modules: [path.join(__dirname, 'node_modules')],
  alias: {
    dhtmlxgantt: path.join(__dirname, '/node_modules/dhtmlx-gantt/codebase/dhtmlxgantt.css')
  }
};
```

And the import statement of this css file could be updated to:

```css
::ng-deep {
    @import "~dhtmlxgantt";
}
```

Problem solved!


<h5>References</h5>
<ul>
	<li><a href="https://dhtmlx.com/blog/using-dhtmlxscheduler-and-dhtmlxgantt-with-typescript/" target="_blank">dhtmlx blog: Using dhtmlxScheduler and dhtmlxGantt with TypeScript</a></li>
	<li><a href="https://dhtmlx.com/blog/dhtmlx-gantt-chart-usage-angularjs-2-framework/" target="_blank">dhtmlx blog: DHTMLX Gantt Chart Usage with Angular 2 Framework</a></li>
	<li><a href="https://hackernoon.com/how-to-use-javascript-libraries-in-angular-2-apps-ff274ba601af" target="_blank">Hackernoon: How to use JavaScript libraries in Angular 2+ apps</a></li>
	<li><a href="https://angular.io/guide/component-styles" target="_blank">Angular: Component Styles</a></li>
	<li><a href="https://hackernoon.com/the-new-angular-ng-deep-and-the-shadow-piercing-combinators-deep-and-drop-4b088dbe459" target="_blank">Hackernoon: The New Angular ::ng-deep and the Shadow-Piercing Combinators Drop</a></li>

</ul>
