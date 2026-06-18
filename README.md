# continuous-artifact-browser

Ideology: The web was built so that you could locate resources. The browser was built to render these resources. Once this was established the browser expanded its capability surface to allow for user actions. Once the user surface included actions, entities started to influence the actions that users took. This became the driving machine for how the web developed. 

However, this machine has rotted the web. 

## LLMs

Once large AI models entered the picture, a new development occurred. There was more incentive than ever to have: one box, any question or request, the model handles it based on emergent knowledge from the 

However, this was always going to be geared towards B2B for a few reasons:
- Consumer technology creates more non-determinism. Heterogeneity across people + their tasks. Businesses have SOPs + 

# MVP
A ton of installable browser extensions on Chrome.

- artifact-jumpstart: The idea is that for things we have _done before_ or projects where you have a high-level sense of _what needs to be done_. The concrete form is that we sometimes have a sense of the artifacts that will be generated once we are done. Maybe it would be a good idea to help jumpstart that artifact (or continue progress)

- mcp-shadows: Every website with an MCP configured could be exposed through a UI. This UI could be configurable or not. But regardless for more complex operations where the user is likely to want a how-to guide, they can generate one on the fly via the information in the MCP. If the page does not have an MCP then this defaults to being able to query the page content to 
> Lots of requirements: centralized auth service, index of MCPs, likely want a passthrough server to store

- [WEAK] personal-search-index: You have pages that are important to you! Search engines do not surface those pages (e.g. your LinkedIn profile, or your most viewed Github page).
> This is somewhat a weak argument as most modern browsers do show pages that are important to you when you are typing in the URL bar. But that URL bar is different from the search bar of a search engine (I am sure for good reason in most cases). 

- tab-completion: Tab already moves you around components on the page (related to / defined by the Accessibility tree in the DOM). What if it made an intelligent guess about where you want to go / the next few steps you want to take. Then tab completion could become a real thing. 

- more-input-domains: This is a comparison to LLMs. LLMs have decided that they do not need people configuring different input schemas for what happens (although they have exposed this to business-facing customers via JSON input schemas, pipelines, etc.). 

- SERP-style-output: The search engine results page (SERP) are all of the webpages which are exposed to you. In addition, there has been a rise of Zero-Click Search (ZCS) where search engines show you the results immediately (sports scores, quick answers, AI summary answers, etc.). The idea is that within LLMs, when you generate an output then that output

- auto-save: Often times, I am stuck at point in the process where there isn't a save built in (e.g. making a JIRA ticket). Closing the tab resets the progress. URLs are great at resource _location_ but now temporally I have taken other actions and I want to save everything so far. Even with Github writing _this document_ I ran into an issue where to decluter I closed all of my tabs (and my progress here was reset). Agent's could easily save what has happened so far for easy restoration.

- agentic-urls: URLs that trigger the agent to do things automatically upon page load.
> Advertisers would love to get their hands on this

- smart-search-history: Querying search history is terrible but people do use it. People do want to remember the places they have been but they want so much more. There are terms that are meaningful to them that they want. They would love to know how much time they spent on Facebook or Youtube this week, etc. This information is all there but is terrible to query. This is along the lines of how an MCP shadow for a page but now introducing the temporal axis. 





