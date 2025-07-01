# Automated Document Constructor

This GitHub Action automates the process of transforming your plain Markdown files into beautifully formatted PDF documents, leveraging the power of **Pandoc** for conversion and **LaTeX** for high-quality typesetting.

Perfect for:
* Generating user manuals.
* Creating structured reports from simple text.
* Maintaining consistent document formatting across versions.
* Anyone who prefers writing in Markdown but needs a PDF output.

Got questions or need assistance? Feel free to [open an issue](https://github.com/yaouDev/automated-document-constructor/issues)!

## Getting Started

Follow these simple steps to generate your PDF document:

1.  **Prepare Your Markdown Files:**
    * Place all your Markdown documents (`.md` files) into the `./docs` folder. This includes any Markdown files within sub-folders – the constructor will find and compile them all.
    * Ensure your Markdown is clean and adheres to standard Pandoc Markdown syntax for best results.

2.  **Trigger the Build Workflow:**
    * **Automatic:** The `Build PDF` workflow will automatically run whenever you push changes to any file within the `./docs` folder on your `main` branch.
    * **Manual:** You can also trigger the build manually at any time. Go to the "Actions" tab in this repository, select the "Build PDF" workflow from the left sidebar, and click "Run workflow".

3.  **Access Your New Document:**
    * Once the GitHub Action completes successfully (typically within ~2 minutes), navigate to the "Actions" tab.
    * Click on the specific workflow run you just executed.
    * Scroll down to the "Artifacts" section. Here you will find the generated PDF files available for download as `.zip` archives. Look for artifacts like your versioned PDF (e.g., `manual-YYYY-MM-DD.pdf`) and the `manual.pdf` (latest version).
    **OR**
    * Navigate to `./artifacts/`

### Including Images

To seamlessly integrate images into your PDF:
- Add your image files (e.g., `.png`, `.jpg`, `.gif`) to the `./images` folder.
- In your Markdown document, use the following syntax where you want the image to appear:
  ```markdown
  ![<figure text or leave empty>](images/<your image file name including file ending>)
  ```
  *Example: `![Diagram of the workflow](images/workflow-diagram.png)`*

## Default Configurations

- **Artifacts Location:** All produced PDF artifacts will be placed within the `./artifacts/versions` folder in your repository, with the very latest version also being put in `./artifacts` (e.g., `manual.pdf`).
    - The base artifact directory (`artifacts` in this example) can be easily changed by updating the `ARTIFACT_DIR` repository variable in your repository settings.
- **Workflow Triggers:** An artifact (PDF) will be built automatically anytime changes are pushed to the `./docs` folder on the `main` branch. You can also manually trigger the `Build PDF` workflow from the "Actions" page in your repository.
- **Versioned Outputs:** Each new PDF generated will include a timestamp for unique identification (e.g., `manual-2025-07-01.pdf`). These will appear in the `./artifacts/versions` folder.
- **Cleanup Workflow:** A dedicated workflow (`Clean Artifacts Versions Folder`) is set up to clean the `./artifacts/versions` folder. Before deletion, it will create a `.zip` archive of all files in that folder and make it available as a workflow artifact for 14 days on the Actions page. This provides a safety net for historical versions.
- **Default Name:** The default base name for your documents ("manual") can be changed by editing the `default` variable of the `file_base_name` block in the workflow_call section within the `build_pdf.yml` file.
- **Document Sorting:** Documents are sorted and compiled lexicographically ascending based on their file names. This means using a structured naming convention (e.g., `010-Introduction.md`, `020-Chapter-One.md`) is crucial for logical compilation order, especially when using sub-folders (e.g., `01-Section/010-Page.md`).

## Advanced Customization

### LaTeX Template
This workflow uses a custom LaTeX template located at `./templates/template.tex` for PDF generation. You can modify this template to customize the PDF's styling, headers, footers, table of contents appearance, and more.

## Recommendations

- Use a multiple digit number prefix to label your documents, e.g `100`, as this will allow index injection for additional documents
- If possible, use sub-folders to keep your chapters structured
- The workflow automatically commits the generated PDF(s) to your repository. These commits include the tag `[skip ci]` in their message. This tag is crucial to prevent the workflow from triggering itself in an infinite loop after pushing new artifacts. Do not remove this tag if you modify the commit message.
- The title page **must** be parsed first

## How to Integrate this as a Document Submodule

If you manage your project's documentation in a separate Git submodule (e.g., in a `docs-source/` folder) and want to use this repository's automated PDF generation for that content, you'll utilize its **reusable workflow** feature.

This means the workflow definition itself (`build_pdf.yml`) resides in *this* repository, and your project's main repository will call it. You **won't** place the `build_pdf.yml` directly inside your submodule's `.github/workflows/` folder, as GitHub Actions doesn't discover workflows there.

Instead, in your project's main repository, create a workflow (e.g., `.github/workflows/build_project_docs.yml`) like this:

```
name: Build Project Documentation PDF

on:
  push:
    branches:
      - main
    paths:
      - 'docs-source/**' # customize this to trigger whenever you push a documentation update
  workflow_dispatch:

jobs:
  build_docs:
    runs-on: ubuntu-latest

    permissions:
      contents: write # needed for checkout and potentially for committing artifacts back

    env:
        FILE_BASE_NAME: my-project-manual # this is your default name
        SUBMODULE_DIR: docs-source # this is the name of your folder - make it match

    steps:
      - name: Checkout main repository
        uses: actions/checkout@v4
        with:
          # if the submodule contains your .md files and images, you must recursively check it out.
          # if it's a private submodule, ensure your GITHUB_TOKEN has permission or use an SSH key.
          submodules: recursive 
          fetch-depth: 0 # fetch full history

      - name: Call Document Constructor Reusable Workflow
        uses: yaouDev/automated-document-constructor/.github/workflows/build_pdf.yml@main # this is the call to the original build workflow, DO NOT change
        with:
          # pass inputs to the reusable workflow
          file_base_name: ${{ env.FILE_BASE_NAME }} # override default 'manual' with your local environment variable
          # assume your reusable workflow has an input for source_docs_path
          source_docs_path: './${{ env.SUBMODULE_DIR }}/docs' # assuming docs are in 'docs-source/docs' inside the submodule
          source_images_path: './${{ env.SUBMODULE_DIR }}/images' # assuming images are in 'docs-source/images' inside the submodule

        id: build_doc_output # give an ID to capture outputs from the reusable workflow

      - name: Use PDF Path from Reusable Workflow
        run: |
          echo "PDF was generated at: ${{ steps.build_doc_output.outputs.pdf_path }}"
          # you could then use this path to upload the PDF as an artifact specific to this project's workflow
          # or integrate it further.

      - name: Upload Generated PDF as Project Artifact (Optional)
        # this uploads the PDF as an artifact for *this* calling workflow run.
        if: success() && steps.build_doc_output.outputs.pdf_path != ''
        uses: actions/upload-artifact@v4
        with:
          name: ${{ env.FILE_BASE_NAME }}-pdf
          path: ${{ steps.build_doc_output.outputs.pdf_path }} # this captures the output from the original file
          retention-days: 30
```

To add the repository as a sub-module use this command, replacing docs-source to whatever you want the folder to be called:

`git submodule add https://github.com/yaouDev/automated-document-constructor.git docs-source`

Use this command to keep the sub-module updated:

`git submodule update --remote --init --recursive`

## License

This project is licensed under the [MIT License](LICENSE) - see the [LICENSE](LICENSE) file for details.

## Useful links

- [Markdown Cheat Sheet](https://www.markdownguide.org/cheat-sheet/)
- [Pandoc Manual](https://pandoc.org/MANUAL.html)