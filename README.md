# ITIPS
Index Cooperative Technical Improvement Proposals - Propose smart contract improvements and track development process for IIPs from specification to deployment.

# Introduction
This repo contains the formalized process for adding smart contract features to Index Cooperative products or processes. The intention is for this repo to contain all information about the progress of new features as they go from ideation, to specification, to implementation, and deployment. Features can reference an open IIP or be opened in preparation for an IIP Technical features may include:
- new Manager contracts
- new Operational contracts (i.e. vesting)
- any other feature that may require IC governance to add

# Repo Setup
- All ITIPS past and present can be found in the `ITIPS` folder
- Any assets for a given ITIP can be found in the `assets` folder
- The Issues tab contains potential ideas that have not been formalized into a ITIP

# Contributing
1. Review `ITIP-TEMPLATE.md` to familiarize yourself with the ITIP process
2. Fork the repository by clicking "Fork" in the top right. 
3. Add your STIP to your fork of the repository. There is a [template STIP here](./STIP-process.md).
4. Submit a pull request in this repository

It's important to follow the `ITIP Template` guidelines, when first opened, an ITIP should only include a summary of the problem/feature and any background information relevant to the problem. If any images or graphs need to be added they can be added to the `assets/STIP-xx` folder where `xx` is the STIP's assigned number.

For tracking purposes it's important that each step of the ITIP is merged sequentially, that's not to say you can't work ahead locally, but we want to be sure all concerned parties have been able to review and approve every step along the way.

# Using Markdown
Most IDEs have options for previewing the generated `.md` file locally:
- [VisualStudio](https://code.visualstudio.com/docs/languages/markdown)
- [Sublime](https://packagecontrol.io/packages/MarkdownLivePreview)

Additional resources for working with Markdown:
- [Basics of Markdown](https://www.markdownguide.org/basic-syntax/)
- [Markdown Tables](https://www.markdownguide.org/extended-syntax/)
- [Markdown Table Generator](https://www.tablesgenerator.com/markdown_tables)

