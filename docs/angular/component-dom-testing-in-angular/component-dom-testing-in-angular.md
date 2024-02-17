Article on Medium: https://medium.com/@borzifrancesco/component-dom-testing-in-angular-0d2256414c06

What is Component DOM testing?
==============================

Components consist of a [TypeScript class](https://www.typescriptlang.org/docs/handbook/2/classes.html) and an [HTML template](https://angular.dev/guide/templates) associated with it. Doing DOM Testing means that we test the Component as a whole entity, instead of testing only the Component’s class.

By doing this, we not only test the Component's HTML code but also **make our test behave in a way that better simulates the interaction that a real user would have with the Component**.

The Angular framework provides powerful, often overlooked [tooling](https://angular.dev/guide/testing/components-basics#component-dom-testing) to perform Component DOM Testing as part of the app’s **unit tests suite**. [Real code examples will be shown later in this article](#09e5).

Component DOM testing and e2e tests
===================================

_By e2e (end-to-end) tests, I mean tests performed using tools such as_ [_Playwright_](https://playwright.dev/)_,_ [_Cypress_](https://www.cypress.io/)_,_ [_Puppeteer_](https://pptr.dev/)_,_ [_Selenium_](https://www.selenium.dev/)_, etc… that test your application as a whole instead of checking single units, as is done in unit tests._

Sometimes people wonder:

> Why should I test the HTML code in the unit tests? Isn’t that a job for the e2e tests?

Let me be clear on one thing: component DOM tests performed at unit level **cannot completely replace e2e tests**, because there will always be scenarios where you need proper e2e tests in place to cover certain scenarios. However, **having Component DOM tests in place can tremendously help reduce the effort needed in e2e tests**.

In an ideal scenario, e2e tests should only consist of a small subset of the component unit tests, as well as covering scenarios that are not possible to cover during unit tests _(more about this later in this article)_.

This aligns with the principle of the [**Automation Testing Pyramid**](https://en.wikipedia.org/wiki/Test_automation#Testing_at_different_levels)**:**

graphical representation of an Automation Test Pyramid ([source](https://martinfowler.com/articles/practical-test-pyramid.html)).

> The “Automation Testing Pyramid” strategy is a testing approach that emphasizes having a larger number of lower-level, faster, and more focused automated tests (like unit tests) at the base, with progressively fewer, broader, and slower tests (like integration and end-to-end tests) as you move up the pyramid.

Discussing the details and benefits of the Testing Pyramid strategy goes beyond the scope of this article. If this is the first time you heard about it, consider reading more about this topic.

Benefits of Component DOM Testing
=================================

Component DOM tests present several advantages **compared to e2e tests**:

1\. Faster
----------

Component DOM tests running as part of the unit tests suite (which is `ng test` for Angular apps) are significantly faster than e2e tests and require fewer dependencies and less setup time. Adding too many e2e tests typically leads to an ever-growing execution time of e2e on the CI, degrading the overall developer experience.

2\. Stability
-------------

I’m sure any experienced frontend developer had to deal at least once in a lifetime with flaky e2e tests (i.e. tests that fail inconsistently). When dealing with Component DOM tests, the chance of getting flaky tests is substantially lower.

3\. Better diagnostic
---------------------

When an e2e test fails, it is not always obvious to determine which part of the code is responsible for the bug causing the issue. In contrast, with Component DOM tests — assuming they are properly written and the Component’s dependencies are correctly mocked — it becomes instantly visible which component is causing the failure.

Component DOM tests have also advantages **compared to plain unit tests:**

4\. Reliability
---------------

The goal of automated tests is to inform us if something no longer works. **A good automated test consistently fails when something is broken.** In other words, if you can break your app and your tests still pass, they are not effective tests, right?

Now, the Component’s template is written in HTML code. When working on it, bugs can accidentally be introduced just like any other kind of code. If your unit tests test your components by directly accessing the Component’s class, they will never be able to catch bugs introduced in the HTML code. You could even delete the whole HTML, and they would still pass. If your unit tests perform proper Component DOM tests, then they will be able to catch failures when your component is broken because of a bug in the HTML template.

5\. Bonus: better code architecture
-----------------------------------

This point might not be obvious at first glance, so I believe it deserves some extra attention. When testing components by accessing them via their template rather than directly accessing their component class, you indirectly encourage yourself to write better code in the first place.

If you pay attention to your test coverage, you might find it sometimes challenging to cover some parts of your component’s logic. When this happens, chances are that the architecture of your component is not optimal. For example, **you might have too much logic in your component and some of it should be moved to a service**.

In some cases, class members are made **public** only to facilitate unit test access. However, this practice has its drawbacks, as a component’s public API should ideally be limited to its template (more about this later in the article, with a [code example](#a48c)).

Another example is when there are cases where you find it easier to fulfil your coverage requirements by directly accessing the component’s class instead of going through the component’s template. In such cases, chances are that the architecture of your component is too complex and should probably be split into smaller components.

As it is well known, it is recommended to keep your components as small and “dumb” as possible. **If writing the component’s DOM tests feels challenging, your component architecture should probably be improved.**

_By the way, I have written an article about test coverage [here](https://medium.com/@borzifrancesco/why-i-set-my-unit-test-coverage-threshold-to-100-4c7138276053?source=post_page-----0d2256414c06--------------------------------)

For the aforementioned reasons and in line with the Testing Pyramid principles, in my opinion, I would assert that: **everything that can be tested at a lower level (i.e. unit tests), should be tested at a lower level**.

Nevertheless, I want to emphasize that Component DOM tests are not meant to completely replace the e2e tests, but they indeed help reduce the number of e2e tests needed.

Now enough talking… Let’s do some coding!

Minimal example: the counter demo app
=====================================

For demo purposes, I have created a minimal application featuring a simple counter. The requirements of such an app are the following:

*   **show** a numeric counter with the default number 0
*   allow the user to **increase** the counter by clicking on an “Increase” button
*   allow the user to **decrease** the counter by clicking on a “Decrease” button, without allowing negative numbers
*   allow the user to **reset** the counter value to 0 by clicking on a “Reset” button

Our counter demo app running

The counter’s Component class
-----------------------------

```
@Component({  
  selector: 'testing-demo-app-counter',  
  standalone: true,  
  changeDetection: ChangeDetectionStrategy.OnPush,  
  imports: \[CommonModule\],  
  templateUrl: './counter.component.html',  
  styleUrl: './counter.component.scss',  
})  
export class CounterComponent {  
  count = 0;  
  
  increase(): void {  
    this.count++;  
  }  
  
  decrease(): void {  
    if (this.count > 0) {  
      this.count--;  
    }  
  }  
  
  reset(): void {  
    this.count = 0;  
  }  
}
```

The counter’s Component template
--------------------------------

```
<div class="main-wrapper">  
  <button data-test-id="reset" (click)="reset()">Reset</button>  
  <button data-test-id="decrease" (click)="decrease()">Decrease</button>  
  <button data-test-id="increase" (click)="increase()">Increase</button>  
  <span data-test-id="count">{{ count }}</span>  
</div>
```

Testing the CounterComponent without DOM tests
----------------------------------------------

This is how we would test the _CounterComponent_ by directly accessing its TypeScript class:

```
// CounterComponent - Component class testing  
  
beforeEach(async () => {  
  await TestBed.configureTestingModule({  
    imports: \[CounterComponent\],  
  }).compileComponents();  
});  
  
function setup() {  
  const fixture = TestBed.createComponent(CounterComponent);  
  const component = fixture.componentInstance;  
  return { fixture, component };  
}  
  
it('should have the counter set to 0 by default', () => {  
  const { component } = setup();  
  
  expect(component.count).toBe(0);  
});  
  
it('should increase the counter when clicking on the increase button', () => {  
  const { component } = setup();  
  
  component.increase();  
  
  expect(component.count).toBe(1);  
});  
  
it('should decrease the counter if the current value is greater than 0 when clicking on the decrease button', () => {  
  const { component } = setup();  
  component.count = 2;  
  
  component.decrease();  
  
  expect(component.count).toBe(1);  
});  
  
it('should NOT decrease the counter if the current value is 0 when clicking on the decrease button', () => {  
  const { component } = setup();  
  component.count = 0;  
  
  component.decrease();  
  
  expect(component.count).toBe(0);  
});  
  
it('should reset the counter when clicking the reset button', () => {  
  const { component } = setup();  
  component.count = 123;  
  
  component.reset();  
  
  expect(component.count).toBe(0);  
});
```

Some notes about the above code:

*   I’m following the **AAA convention** (**_A_**_rrange,_ **_A_**_ct, and_ **_A_**_ssert_ statements separated by a blank line in every spec)
*   I’m wrapping the creation of elements like the **_fixture_** and the **_component_** inside a **_setup()_** function in order to avoid shared variables across specs

These tests work and even provide us with [100% coverage](https://medium.com/@borzifrancesco/why-i-set-my-unit-test-coverage-threshold-to-100-4c7138276053), but as you can guess they are not that effective. If bugs are introduced in the Component’s template (i.e. the HTML code), these tests would not catch them at all.

Actually, I could delete the whole HTML code and these tests would still report: _all good, all green, everything is working fine!_ And that would be a significant oversight, wouldn’t it?

Writing DOM tests for our CounterComponent
------------------------------------------

Before writing the DOM tests for our CounterComponent, let’s first make a small change in our Component’s template:

```
<div class="main-wrapper">  
  <button data-test-id="reset" (click)="reset()">Reset</button>  
  <button data-test-id="decrease" (click)="decrease()">Decrease</button>  
  <button data-test-id="increase" (click)="increase()">Increase</button>  
  <span data-test-id="count">{{ count }}</span>  
</div>
```

All I did here was add a **data-test-id** attribute to the HTML elements of my template, in order to use them from the automated tests to uniquely access the elements.

Technically, you could also use _CSS classes_ or _CSS IDs_, that is entirely up to you. I normally like using data-test-id attributes so I know that those attributes are merely used for the automated tests. And I use them for my e2e tests as well. This is just a personal preference, and you can use whatever convention suits you the best.

To access the HTML elements from our tests, we will use the `fixture` returned by the Angular [TestBed](https://angular.dev/api/core/testing/TestBed) by using:

```
fixture.debugElement.query(By.css(\`SOME\_CSS\_SELECTOR\`)).nativeElement
```

Replacing `SOME_CSS_SELECTOR` with the actual selector, e.g. `[data-test-id="count"]`.

Now, this is a **preliminary version** (which we will improve in a moment) of how the DOM tests of our _CounterComponent_ will look like:

```
// CounterComponent - Component DOM testing  
  
beforeEach(async () => {  
  await TestBed.configureTestingModule({  
    imports: \[CounterComponent\],  
  }).compileComponents();  
});  
  
function setup() {  
  const fixture = TestBed.createComponent(CounterComponent);  
  const component = fixture.componentInstance;  
  return { fixture, component };  
}  
  
it('should have the counter set to 0 by default', () => {  
  const { fixture } = setup();  
  
  fixture.detectChanges();  
  
  expect(fixture.debugElement.query(By.css(\`\[data-test-id="count"\]\`)).nativeElement.innerHTML).toEqual('0');  
});  
  
it('should increase the counter when clicking on the increase button', () => {  
  const { fixture } = setup();  
  fixture.detectChanges();  
  
  fixture.debugElement.query(By.css(\`\[data-test-id="increase"\]\`)).nativeElement.click();  
  fixture.detectChanges();  
  
  expect(fixture.debugElement.query(By.css(\`\[data-test-id="count"\]\`)).nativeElement.innerHTML).toEqual('1');  
});  
  
it('should decrease the counter if the current value is greater than 0 when clicking on the decrease button', () => {  
  const { fixture } = setup();  
  fixture.debugElement.query(By.css(\`\[data-test-id="increase"\]\`)).nativeElement.click();  
  fixture.debugElement.query(By.css(\`\[data-test-id="increase"\]\`)).nativeElement.click();  
  
  fixture.debugElement.query(By.css(\`\[data-test-id="decrease"\]\`)).nativeElement.click();  
  fixture.detectChanges();  
  
  expect(fixture.debugElement.query(By.css(\`\[data-test-id="count"\]\`)).nativeElement.innerHTML).toEqual('1');  
});  
  
it('should NOT decrease the counter if the current value is 0 when clicking on the decrease button', () => {  
  const { fixture } = setup();  
  
  fixture.debugElement.query(By.css(\`\[data-test-id="decrease"\]\`)).nativeElement.click();  
  fixture.detectChanges();  
  
  expect(fixture.debugElement.query(By.css(\`\[data-test-id="count"\]\`)).nativeElement.innerHTML).toEqual('0');  
});  
  
it('should reset the counter when clicking the reset button', () => {  
  const { fixture } = setup();  
  fixture.debugElement.query(By.css(\`\[data-test-id="increase"\]\`)).nativeElement.click();  
  fixture.debugElement.query(By.css(\`\[data-test-id="increase"\]\`)).nativeElement.click();  
  
  fixture.debugElement.query(By.css(\`\[data-test-id="reset"\]\`)).nativeElement.click();  
  fixture.detectChanges();  
  
  expect(fixture.debugElement.query(By.css(\`\[data-test-id="count"\]\`)).nativeElement.innerHTML).toEqual('0');  
});
```

These tests work well and, not only do they guarantee full code coverage of the Component’s TypeScript, but they also make sure that they would fail if we introduce any bug inside the HTML code. **They interact with our component in a similar way that a real user would.**

In case you are wondering about the performance of these tests compared to pure Component class tests, I’m happy to show that there is no sign of performance degradation:

Component class vs DOM testing: performance comparison

There seems to be no significant difference in the execution time between the Component class tests vs Component DOM tests. In other words, there seem to be no real benefits of Component class tests over Component DOM tests.

But we are not done yet. Several things can be improved in our CounterComponent’s DOM tests:

1\. There is too much code repetition
-------------------------------------

Lines such as:

```
fixture.debugElement.query(By.css(\`\[data-test-id="count"\]\`))
```

are repeated over and over. Also if, for some reason, the `data-test-id`of an element should change, there are different places of the test where it should be updated.

2\. Not friendly ‘element not found’ errors
-------------------------------------------

If we make a typo in the CSS selector or if, for some reason, our test at some point expects an element that is currently not shown (e.g. the element has an `*ngIf` with a broken logic), the resulting error would be something like:

> TypeError: Cannot read properties of null (reading ‘nativeElement’)

The reason for this failure might not be obvious.

3\. Mixed logic
---------------

The logic that fetches HTML elements is mixed with the logic of the specs, this is bad and goes against the Single Responsibility Principle (SRP).

We can fix all of the above issues by introducing the **Page Object** design pattern.

The Page Object design pattern
==============================

If you worked with e2e tests before you probably heard of the Page Object pattern. We can use the same pattern for our Component DOM tests.

While discussing the benefits of this pattern goes beyond the scope of this article, it is important to mention that by using page objects, we create a new level of abstraction and separate the logic responsible for retrieving and dealing with HTML elements from the logic of the single tests, in line with the [SOLID principles](https://en.wikipedia.org/wiki/SOLID).

There are several ways we can implement this design pattern and I’m going to show how I typically use it in my Angular apps.

The PageObject abstract class
-----------------------------

This is an abstract class I typically create in every project and then customise according to every project’s specific needs. Here’s a minimal implementation which is suitable for our Counter app use case:

```
abstract class PageObject<ComponentType> {  
  constructor(protected fixture: ComponentFixture<ComponentType>) {}  
  
  detectChanges(): void {  
    this.fixture.detectChanges();  
  }  
  
  getDebugElementByCss(cssSelector: string, assert = true): DebugElement {  
    const debugElement = this.fixture.debugElement.query(By.css(cssSelector));  
  
    if (assert) {  
      // Jest version (using try-catch):  
      try {  
        expect(debugElement).toBeTruthy();  
      } catch(e) {  
        throw new Error(\`Element with selector "${cssSelector}" was not found.\`);  
      }  
      // Jasmine version (one-liner):  
      // expect(debugElement).withContext(\`Element with selector "${cssSelector}" was not found.\`).toBeTruthy();  
    }  
  
    return debugElement;  
  }  
  getDebugElementByTestId(testId: string, assert = true): DebugElement {  
    return this.getDebugElementByCss(\`\[data-test-id="${testId}"\]\`, assert);  
  }  
}
```

The `PageObject` is an abstract class taking a [generic type](https://www.typescriptlang.org/docs/handbook/2/generics.html) parameter `ComponentType` which determines the type argument of the `fixture`passed to the constructor. This usage helps enforce better type hints for the fixture (which is of type [ComponentFixture](https://angular.dev/api/core/testing/ComponentFixture)).

The abstract class has three methods:

*   **detectChanges**: just exposes `fixture.detectChanges()` ;
*   **getDebugElementByCss:** takes a `cssSelector` as input and returns a [DebugElement](https://angular.dev/api/core/DebugElement). By default, it fails with a developer-friendly error if the element is not found. An optional`assert` boolean input, defaulting to`true`, determines whether it should fail with an error when the element represented by the `cssSelector` is not found. This is useful when expecting that a certain element is NOT present in the DOM under specific conditions, for instance, when testing an element with an`*ngIf` statement that is false;
*   **getDebugElementByTestId**: this is just a shortcut to get elements by their `data-test-id` attributes;

_I then typically add more logic to the abstract class PageObject according to the project needs…_

The CounterComponent page object class
--------------------------------------

Now it is time to extend the `PageObject` abstract class to create a specific page object class for our CounterComponent:

```
class CounterPage extends PageObject<CounterComponent> {  
  // methods to retrieve the elements  
  getIncreaseButton(): DebugElement {  
    return this.getDebugElementByTestId('increase');  
  }  
  getDecreaseButton(): DebugElement {  
    return this.getDebugElementByTestId('decrease');  
  }  
  getResetButton(): DebugElement {  
    return this.getDebugElementByTestId('reset');  
  }  
  getCount(): DebugElement {  
    return this.getDebugElementByTestId('count');  
  }  
  
  // methods to perform actions  
  clickIncreaseButton(): void {  
    this.getIncreaseButton().nativeElement.click();  
  }  
  clickDecreaseButton(): void {  
    this.getDecreaseButton().nativeElement.click();  
  }  
  clickResetButton(): void {  
    this.getResetButton().nativeElement.click();  
  }  
  expectCurrentCountToBe(value: number) {  
    expect(this.getCount().nativeElement.innerHTML).toEqual(\`${value}\`);  
  }  
}
```

The `CounterPage` class encapsulates the logic to access the DOM and perform certain actions, this new level of abstraction allows the specs to focus on higher-level concerns without dealing with the DOM manipulation aspect.

Let’s now see it in action in our _CounterComponent_ tests.

CounterComponent tests: final version
-------------------------------------

This is an improved version of our DOM component tests example that uses the Page Object design pattern. The test specs are now simpler, and more readable.

```
beforeEach(async () => {  
  await TestBed.configureTestingModule({  
    imports: \[CounterComponent\],  
  }).compileComponents();  
});  
  
function setup() {  
  const fixture = TestBed.createComponent(CounterComponent);  
  const page = new CounterPage(fixture); // <- here we create the page object  
  return { page };  
}  
  
it('should have the counter set to 0 by default', () => {  
  const { page } = setup();  
  
  page.detectChanges();  
  
  page.expectCurrentCountToBe(0);  
});  
  
it('should increase the counter when clicking on the increase button', () => {  
  const { page } = setup();  
  
  page.clickIncreaseButton();  
  page.detectChanges();  
  
  page.expectCurrentCountToBe(1);  
});  
  
it('should decrease the counter if the current value is greater than 0 when clicking on the decrease button', () => {  
  const { page } = setup();  
  page.clickIncreaseButton();  
  page.clickIncreaseButton();  
  
  page.clickDecreaseButton();  
  page.detectChanges();  
  
  page.expectCurrentCountToBe(1);  
});  
  
it('should NOT decrease the counter if the current value is 0 when clicking on the decrease button', () => {  
  const { page } = setup();  
  
  page.clickDecreaseButton();  
  page.detectChanges();  
  
  page.expectCurrentCountToBe(0);  
});  
  
it('should reset the counter when clicking the reset button', () => {  
  const { page } = setup();  
  page.clickIncreaseButton();  
  page.clickIncreaseButton();  
  
  page.clickResetButton();  
  page.detectChanges();  
  
  page.expectCurrentCountToBe(0);  
});
```

Improved Component’s class
--------------------------

Since [Angular 14](https://blog.angular.io/angular-v14-is-now-available-391a6db736af), the Component’s class members (methods and attributes) no longer require to be `public`in order to be accessed by the Component’s template. So we no longer need the Component class members to be `public` .

Attributes and methods of the Component’s class should always be:

*   `protected` when they should be accessed by the template
*   `private` otherwise

We can now refactor our _CounterComponent_ class and it will look like this:

```
@Component({  
  selector: 'testing-demo-app-counter',  
  standalone: true,  
  changeDetection: ChangeDetectionStrategy.OnPush,  
  imports: \[CommonModule\],  
  templateUrl: './counter.component.html',  
  styleUrl: './counter.component.scss',  
})  
export class CounterComponent {  
  protected count = 0;  
  
  protected increase(): void {  
    this.count++;  
  }  
  
  protected decrease(): void {  
    if (this.count > 0) {  
      this.count--;  
    }  
  }  
  
  protected reset(): void {  
    this.count = 0;  
  }  
}
```

_You can find the entire source code of the counter demo app_ [_here_](https://github.com/FrancescoBorzi/testing-demo-app/tree/main/libs/counter)_._

Conclusions
===========

*   Testing Components by accessing their template allows us to write better tests that are **more effective** and **better simulate the way real users would interact with them**;
*   The Angular framework already provides great tooling to write efficient Component DOM tests without the need to import any external library;
*   Component DOM tests are not meant to replace e2e tests, but **reduce the number of e2e tests needed**;
*   When certain scenarios can be covered in our unit tests suite using the Component DOM testing technique, they should. Because **they run faster and are less flaky compared to e2e tests**. Also, they can pinpoint the failure more easily;
*   Testing components by accessing their templates results in **more reliable unit tests** which, as a side effect, sometimes encourage us to write a **better code architecture**, compared to testing components by accessing directly their class;
*   Using the **Page Object design pattern allows to separate the DOM manipulation logic from the test logic**, resulting in specs that avoid code repetitions and are much simpler, readable and easier to be maintained.

Thanks
------

I would like to thank the people who helped me by reviewing this article:

*   [Stefanos Lignos](https://medium.com/@stefanoslignos) ([LinkedIn](https://www.linkedin.com/in/stefanoslignos/), [GitHub](https://github.com/stefanoslig))
*   [Femke Schripsema](https://medium.com/@femkemarije) ([LinkedIn](https://www.linkedin.com/in/femkeschripsema/))
