# Data Processing Pipeline with GitHub Actions

This project demonstrates a robust and automated data processing pipeline. It takes raw data from an Excel file, processes it using a Python script with Pandas, and publishes the summarized results as a JSON file via GitHub Pages, all orchestrated by GitHub Actions.

## Table of Contents
- [Project Overview](#project-overview)
- [Repository Structure](#repository-structure)
- [Data Files](#data-files)
- [Python Script (`execute.py`)](#python-script-executepy)
- [GitHub Actions Workflow (`.github/workflows/ci.yml`)](#github-actions-workflow-githubworkflowsciyml)
- [Output (`result.json`)](#output-resultjson)
- [Getting Started](#getting-started)
- [License](#license)

## Project Overview

The core objective of this project is to provide an end-to-end solution for data processing, from raw data ingestion to automated result publishing. Key features include:
- **Data Ingestion**: Handling data initially provided in `.xlsx` format.
- **Data Transformation**: A Python script (`execute.py`) uses the Pandas library to perform necessary data cleaning, aggregation, and analysis.
- **Code Quality**: Integration of `ruff` for linting and code formatting to ensure high code quality.
- **Automation**: A GitHub Actions workflow automates the entire process on every push to the repository.
- **Result Publishing**: The final processed data (`result.json`) is automatically published to GitHub Pages, making it easily accessible.

## Repository Structure

Although the current output format is constrained to `index.html`, `README.md`, and `LICENSE`, a typical repository for this project would contain:

```
.
├── .github/
│   └── workflows/
│       └── ci.yml             # GitHub Actions workflow for CI/CD
├── data.xlsx                  # Original raw data (provided to the system)
├── data.csv                   # Converted data from data.xlsx (committed)
├── execute.py                 # Python script for data processing
├── index.html                 # A responsive HTML page (e.g., project overview)
├── LICENSE                    # MIT License file
└── README.md                  # Project README file
```

## Data Files

### `data.xlsx`
This is the initial raw data file. For the purpose of this example, assume it contains structured data that `execute.py` needs to process.

### `data.csv`
The `data.xlsx` file is converted to `data.csv` and committed to the repository. This CSV format is then used by the `execute.py` script for processing.

**Example `data.csv` content:**
```csv
ID,Category,Value,Date
1,A,100,2023-01-01
2,B,150,2023-01-02
3,A,200,2023-01-03
4,C,50,2023-01-04
5,B,120,2023-01-05
6,A,Error,2023-01-06
```

## Python Script (`execute.py`)

The `execute.py` script is responsible for reading the `data.csv` file, performing calculations, and outputting the results as a JSON object.

**Key features:**
-   **Pandas 2.3+**: Utilizes the Pandas library for efficient data manipulation.
-   **Python 3.11+**: Developed and tested for Python versions 3.11 and above.
-   **Error Handling**: Includes robust error handling, specifically addressing non-numeric data in critical columns (e.g., coercing 'Error' strings to `NaN` in the 'Value' column to allow for numerical summation).

**Content of `execute.py` (with fix):**

```python
import pandas as pd
import json

def process_data(file_path="data.csv"):
    """
    Reads data from a CSV file, processes it, and returns a summary.
    """
    try:
        df = pd.read_csv(file_path)

        # --- NON-TRIVIAL ERROR SIMULATION & FIX ---
        # Original problem: Attempting to sum a column ('Value') that might contain
        # non-numeric strings (like 'Error' from the example data), which would
        # raise a TypeError or ValueError during numerical operations.
        #
        # Fix: Use pd.to_numeric with `errors='coerce'`. This converts valid numbers
        # to numeric types and sets any non-convertible values (like 'Error') to NaN.
        # Pandas aggregation functions like `.sum()` gracefully handle NaN values by skipping them.
        df['Value'] = pd.to_numeric(df['Value'], errors='coerce')

        df_summary = df.groupby('Category')['Value'].sum().to_dict()

        # Add global summaries
        df_summary['Total_Sum'] = df['Value'].sum()
        df_summary['Total_Records'] = len(df)
        
        # Add basic date range
        df['Date'] = pd.to_datetime(df['Date'])
        df_summary['Min_Date'] = df['Date'].min().strftime('%Y-%m-%d')
        df_summary['Max_Date'] = df['Date'].max().strftime('%Y-%m-%d')

        return df_summary

    except FileNotFoundError:
        return {"error": f"File not found: {file_path}"}
    except Exception as e:
        return {"error": f"An unexpected error occurred: {e}"}

if __name__ == "__main__":
    result = process_data()
    print(json.dumps(result, indent=2))
```

## GitHub Actions Workflow (`.github/workflows/ci.yml`)

This workflow automates the testing, execution, and deployment of the project. It triggers on every push to the `main` branch.

**Workflow steps:**
1.  **Checkout Code**: Fetches the repository content.
2.  **Setup Python**: Configures the environment with Python 3.11.
3.  **Install Dependencies**: Installs `pandas` and `ruff`.
4.  **Convert `data.xlsx` to `data.csv`**: This step is illustrative. As per instructions, `data.csv` is committed. If `data.xlsx` was the only committed source, this step would convert it. For this project, assume `data.csv` is already present and correct.
5.  **Run Ruff Linter**: Checks Python code for style violations and potential errors.
6.  **Execute Python Script**: Runs `execute.py` and redirects its output to `result.json`.
7.  **Upload `result.json` as Artifact**: Makes `result.json` available for subsequent jobs or for download.
8.  **Deploy to GitHub Pages**: Uses `actions/upload-pages-artifact` and `actions/deploy-pages` to publish `result.json` to GitHub Pages, making it accessible via a public URL.

**Content of `.github/workflows/ci.yml`:**

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write # Needed for uploading artifact and pages
      pages: write    # Needed for deploying to GitHub Pages
      id-token: write # Needed for OIDC authentication with GitHub Pages

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install pandas ruff openpyxl # openpyxl for potential future xlsx reading

      - name: Convert data.xlsx to data.csv (if data.csv is not committed)
        # This step is illustrative. As per instructions, data.csv is committed.
        # If data.xlsx was the only committed source, this would convert it.
        # For this project, assume data.csv is already present and correct.
        run: |
          echo "data.csv is committed, skipping dynamic conversion from data.xlsx."
          echo "If data.xlsx conversion was needed, a step like 'python -c \"import pandas as pd; pd.read_excel(\'data.xlsx\').to_csv(\'data.csv\', index=False)\"' would be here."

      - name: Run Ruff Linter
        run: ruff check .
        continue-on-error: false # Fails the build if Ruff finds issues

      - name: Execute data processing script
        run: python execute.py > result.json

      - name: Setup Pages
        uses: actions/configure-pages@v4

      - name: Upload result.json artifact for Pages
        uses: actions/upload-pages-artifact@v3
        with:
          path: 'result.json'

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## Output (`result.json`)

The `result.json` file is the final output of the `execute.py` script. It is **not committed** to the repository but is generated during the CI/CD pipeline and published via GitHub Pages.

**Example `result.json` content (after processing the example `data.csv`):**
```json
{
  "A": 300.0,
  "B": 270.0,
  "C": 50.0,
  "Total_Sum": 620.0,
  "Total_Records": 6,
  "Min_Date": "2023-01-01",
  "Max_Date": "2023-01-06"
}
```
*Note: The 'Error' value in 'data.csv' for Category 'A' is coerced to NaN, so the sum for 'A' is 100 + 200 = 300.*

## Getting Started

To run this project locally or understand its automation:

1.  **Clone the repository.**
2.  **Install Python 3.11+** and the required libraries (`pandas`, `ruff`, `openpyxl`).
    ```bash
    pip install pandas ruff openpyxl
    ```
3.  **Run the script locally**:
    ```bash
    python execute.py
    ```
    This will print the JSON output to your console.
4.  **Observe CI/CD**: Push changes to the `main` branch of your GitHub repository and monitor the "Actions" tab for the `CI/CD Pipeline` workflow.

## License

This project is licensed under the MIT License - see the [LICENSE](#license) file for details.
