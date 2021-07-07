# Verify Sourcemaps in `npm publish` and `npm pack`

## Summary

Validates sourcemaps of public packages as part of publishing in order to ensure more reliable IDE and browser debugging, especially for novice JS developers who may find it difficult to debug into transpiled code.

## Motivation

Unfortunately, it's very easy to publish a public OSS package with broken sourcemaps.
This degrades the debugging experience in IDEs or browser devtools because only transpiled code (or, in rare cases, the wrong source code!) is shown to the user.
Experienced JS developers may be comfortable debugging into transpiled code, but novice developers can easily get lost. 
This is especially true for complex transpilations like `async`/`await` state machines which are nearly impossible to understand for novices.

Sourcemaps are most commonly broken from the following causes:
- Maintainers may add the `src` folder to `.npmignore`, or may omit it from `files` in `package.json`.
  Sometimes this is because maintainers are passionate about limiting the on-disk size of installed packages on developer workstations.
  Sometimes this is just an oversight.
- A build step may copy output files into a different folder than the sourcemap refers to.
  For example, a transpiler might emit output files into an `out` folder, and a later build step might copy it into another folder like `./dist/lib`.
- Buggy build tools may generate invalid sourcemaps, often only in some options configurations (which explains why the problem wasn't caught earlier).

Sourcemap problems can be hard to spot before publishing because sourcemaps may work properly on the developer's machine (where source files are necessarily present) but not when the package is installed from the registry.

Validating sourcemaps as part of the publishing process would ensure that maintainers won't have to wait until users complain in order to fix broken sourcemaps.

## Detailed Explanation

`npm publish` and `npm pack` validates sourcemaps in the to-be-published package.
By default, validation failures are treated as errors to encourage maintainers to fix sourcemap problems before publishing.

A new `--sourcemap-validation <error|warning|info|off>` allows configuring sourcemap validation as errors (default), warnings, info, or disabling validation completely.

Validation is performed for public packages only.
For `restricted` (private) packages, omitting source code may be a normal occurrence, although see [question](Unresolved-questions-and-bikeshedding) below for discussion.

## Rationale and Alternatives

The current best alternative is to rely on developers who download packages to find and report sourcemap bugs. Examples:
- AWS SDK - https://github.com/aws/aws-sdk-js-v3/pull/1462
- AWS amplify-js - https://github.com/aws-amplify/amplify-js/pull/4216
- immer - https://github.com/immerjs/immer/pull/490
- react-router - https://github.com/ReactTraining/react-router/pull/6823
- swiper - https://github.com/nolimits4web/swiper/pull/3306
- react-confetti - https://github.com/alampros/react-confetti/pull/79
- libphonenumber-js - https://github.com/catamphetamine/libphonenumber-js/pull/306
- popper - https://github.com/popperjs/popper-core/pull/761
- react-window - https://github.com/bvaughn/react-window/pull/275
- fullcalendar - https://github.com/fullcalendar/fullcalendar/pull/4720

Bundler plugins like [`source-map-loader`](https://www.npmjs.com/package/source-map-loader) can help find sourcemap issues by showing console warnings if sourcemaps are invalid.
The problem with relying on post-install bug reporting is that it can only fix the problem after it's been shipped.
Even if a conscientious developer notices the problem and quickly builds a PR, it can take months for PRs to be merged.
Debugging is broken for that package in the meantime.

Another alternative that I've heard from a few maintainers is that developers should be using `console.log` debugging like OG maintainers did before sourcemap-using IDE debuggers existed.
They argue that it's actually a good thing for developers to have to learn how to navigate and understand the "real" transpiled code that's being executed.
This seems like reasonable advice for intermediate-to-advanced developers, but for novices it seems like it'd add an unnecessary obstacle for new JS developers, especially those coming from Java, .NET, or other platforms where IDE debugging is widely used and expected. 

A final alternative is for developers who download packages to work around broken sourcemaps. There are many inconvenient ways to do this:
* IDE configuration like [`sourceMapPathOverrides`](https://github.com/microsoft/vscode-js-debug/blob/47c208ecb0b969ea3de063d691d0e43b54b20677/OPTIONS.md#sourcemappathoverrides-4) in Visual Studio Code
* [patch-package](https://www.npmjs.com/package/patch-package) or similar tool to fix up bad sourcemaps inside `node_modules` after they're downloaded
* custom build steps or bundler plugins that manually fix up sourcemap paths, so that the bundled output's sourcemap is correct even if the input packages' sourcemaps are invalid
* manually copy source files into the correct location in `node_modules` where the source files were supposed to be

What all the alternatives above have in common is that they generate more work for the entire ecosystem, including for library maintainers who have to deal with sourcemap-related bug reports.
Instead, if validation of sourcemaps is baked into publishing, then sourcemap problems can be caught before the buggy sourcemaps are released.
This is easier for developers who can debug more effectively, and easier for maintainers too!

## Implementation

The following validations should be performed on each sourcemap file in the package:
* It's valid JSON
* Parses successfully using Mozilla's [source-map](https://github.com/mozilla/source-map) library
* Source files that are referenced in the sourcemap using relative paths actually exist inside the package.
* Source files that are referenced by absolute file paths should cause an error because the same absolute paths won't exist on the installing developer's computer.
* For source files that are referenced by (non-file) URL, simply verify that the URL is valid. (see [question](Unresolved-questions-and-bikeshedding) below)
* (If not included above) The paths of each source file should not contain characters which would be invalid on some platforms. For example, the `\` character is valid for Windows paths but should not be allowed as a sourcemap path.

Note that validation must happen against the files that will be packaged, not against the files in the folder on disk.
This is important because `.npmignore` and `files` may exclude source files from publishing.

The validations above should be understood to be a minimal set.
It may be good to add additional validations as more sourcemap failure cases are discovered.

At least some of the validations above are already performed by the packages listed in the [Prior Art](#prior-art) section.
These packages may be useful to supply logic, tests, and/or code for validation, for finding all the sourcemap files in a package, for formatting helpful error messages, etc.

Note that both `npm publish` and `npm pack` should get the new features described in this RFC.

## Prior Art

- [`source-map-loader`](https://www.npmjs.com/package/source-map-loader) includes minimal sourcemap validation, and will show build-time console warnings if problems are found.
At a minimum, `npm publish` should include the same validations so that a published package should not generate any warnings from `source-map-loader`.

- [sourcemap-validator](https://www.npmjs.com/package/sourcemap-validator) - this package, as you'd expect from its name, includes some validation of sourcemaps.

- [sourcemaps.io](https://github.com/getsentry/sourcemaps.io) - Code for the https://sourcemaps.io site that is "A re-write of [sourcemap-validator](https://github.com/mattrobenolt/sourcemap-validator) using React, Node, and Google Cloud Functions."

## Unresolved Questions and Bikeshedding

Should we also `fetch` non-file URLs to verify they exist?  This has perf implications, security issues, etc. that I'm not sure we'd want to bite off.

Should validation happen for private packages too?

Are there more options that would be required to control validation behavior?

Is there sufficient documentation for library maintainers to help them fix their sourcemaps?
It would be a bad outcome if these errors caused maintainers to simply turn off sourcemaps in their bundlers.
Ideally, there'd be simple, step-by-step instructions for rollup and other common library bundlers to explain how to fix the problems reported by sourcemap validation.