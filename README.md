# Automated Document Constructor

This GitHub Action automates the process of transforming your plain Markdown files into beautifully formatted PDF documents, leveraging the power of **Pandoc** for conversion and **LaTeX** for high-quality typesetting.

Perfect for:
* Generating user manuals.
* Creating structured reports from simple text.
* Maintaining consistent document formatting across versions.
* Anyone who prefers writing in Markdown but needs a PDF output.

Got questions or need assistance? Feel free to [open an issue](https://github.com/yaouDev/automated-document-constructor/issues)!

## 🚀 Initial Setup Steps

Before you start developing, you need to configure some essential project settings.
I've provided an automated GitHub Actions workflow to help you with this.
The reason it's made this way is so that you can edit your inputs and outputs without creating commits by editting the YAML-file - you can, of course, also change these manually in the settings.

**Please follow these steps:**

1.  **Create a new repository from this template.**
    Click the "Use this template" button above, then choose "Create a new repository".

2.  **Navigate to the Actions tab in your new repository.**
    Once your new repository is created, go to its "Actions" tab.

3.  **Run the "Setup New Repository Variables" workflow.**
    You'll see a workflow named "Setup New Repository Variables" listed. Click on it.

4.  **Click "Run workflow" and provide the required inputs.**
    You will be prompted to enter the values for `output_dir`, `docs_dir`, and `images_dir`.
    Fill these in according to your project's needs.

## Project Structure

* `docs/`: Where your text documents live.
* `images/`: Your project images.
* `artifacts/`: Generated output files.

(The `output_dir`, `docs_dir`, and `images_dir` variables configured by the workflow will reflect these paths.)

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
- **Default Name:** The default base name for your documents ("document-<>") can be changed by editing the `default` variable of the `file_base_name` block in the workflow_call section and the alternative value the environment variable `FILE_BASE_NAME` within the `build_pdf.yml` file.
- **Document Sorting:** Documents are sorted and compiled lexicographically ascending based on their file names. This means using a structured naming convention (e.g., `010-Introduction.md`, `020-Chapter-One.md`) is crucial for logical compilation order, especially when using sub-folders (e.g., `01-Section/010-Page.md`).

## Advanced Customization

### LaTeX Template
This workflow uses a custom LaTeX template located at `./templates/template.tex` for PDF generation. You can modify this template to customize the PDF's styling, headers, footers, table of contents appearance, and more.

## Recommendations

- Use a multiple digit number prefix to label your documents, e.g `100`, as this will allow index injection for additional documents
- If possible, use sub-folders to keep your chapters structured
- The workflow automatically commits the generated PDF(s) to your repository. These commits include the tag `[skip ci]` in their message. This tag is crucial to prevent the workflow from triggering itself in an infinite loop after pushing new artifacts. Do not remove this tag if you modify the commit message.
- The title page **must** be parsed first

## How to Integrate the Automated Document Constructor as a workflow call

Instead of cloning this repository, you're able to make a call to its workflow. Simply create a file such as `.github/workflows/build_pdf.yml` and paste the code below.
You will be prompted to provide some file paths to the directories you are utilizing. Note that you will require at least `images` and `docs` (or your version of them).

```
name: Build PDF

on:
  workflow_dispatch:
    inputs:
      file_base_name:
        required: false
        type: string
        description: 'What your document should be called'
        default: 'my-document'
      output_dir:
        required: false
        type: string
        description: 'Where your documents should end up (if you choose to upload)'
        default: 'my-artifacts'
      docs_dir:
        required: true
        type: string
        description: 'Where your text files are placed'
        default: 'my-documents'
      images_dir:
        required: true
        type: string
        description: 'Where your images are placed'
        default: 'my-images'
      template_path:
        required: false
        type: string
        description: 'Point to your template, "remote_template" will download the default'
        default: 'remote_template'

jobs:
  call_pdf_builder:
    uses: yaouDev/automated-document-constructor/.github/workflows/build_pdf.yml@main
    permissions:
      contents: write

    with:
      file_base_name: ${{ inputs.file_base_name }}
      output_dir: ${{ inputs.output_dir }}
      docs_dir: ${{ inputs.docs_dir }}
      images_dir: ${{ inputs.images_dir }}
      template_path: ${{ inputs.template_path }}

  post_build_actions:
    needs: call_pdf_builder
    runs-on: ubuntu-latest

    steps:
      - name: Display Generated PDF Information
        run: |
          echo "The PDF was generated and is available at: ${{ needs.call_pdf_builder.outputs.pdf_path }}"
          echo "You can view the reusable workflow's artifacts here: ${{ needs.call_pdf_builder.outputs.artifact_url }}"

      - name: Download Generated PDF Artifact
        id: download_pdf
        if: success() && needs.call_pdf_builder.outputs.pdf_path != '' # Corrected job name
        uses: actions/download-artifact@v4
        with:
          name: ${{ needs.call_pdf_builder.outputs.artifact_name }}
          path: ./downloaded_pdf

      - name: Verify Downloaded PDF (Optional)
        if: success() && steps.download_pdf.outputs.download-path != ''
        run: |
          echo "Downloaded PDF to: $(ls -R ./downloaded_pdf)"
```

## License

This project is licensed under the [MIT License](LICENSE) - see the [LICENSE](LICENSE) file for details.

## Useful links

- [Markdown Cheat Sheet](https://www.markdownguide.org/cheat-sheet/)
- [Pandoc Manual](https://pandoc.org/MANUAL.html)
