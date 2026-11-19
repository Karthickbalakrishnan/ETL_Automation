# ETL_Automation

An AI-assisted ETL pipeline for cleaning messy student enrollment data using Python, Pandas, Pydantic, and OpenAI Structured Outputs.

## Project Overview

**Result:** `student_enrollment_raw.csv` is transformed into a clean, validated, deduplicated CSV named `students_cleaned.csv`.

**Task:** Automate the extraction, cleaning, validation, transformation, and export of student enrollment records.

**Challenge:** Raw enrollment files commonly contain inconsistent capitalization, whitespace, invalid email addresses, mixed phone-number formats, inconsistent fee-status values, and dates in formats that are difficult to analyze.

**Framework:** The project uses a repeatable four-step pipeline implemented in `student_etl.py`.

**Recap:** The script reads a raw CSV, sends batches of records to an OpenAI model with a strongly typed Pydantic schema, and exports the cleaned records to CSV.

## Features

- Automatically detects a CSV file when only one candidate exists in the working directory
- Supports an explicit input-file path through the command line
- Reads all values as strings to preserve raw CSV data
- Processes records in batches of 25
- Uses OpenAI Structured Outputs for consistent JSON structures
- Trims and normalizes student names, courses, and cities
- Lowercases email addresses
- Validates standard `user@domain.tld` email formats
- Converts phone numbers to digits only
- Converts fee status values into boolean values
- Converts valid enrollment dates into `YYYY-MM-DD`
- Removes duplicate `Student_ID` values
- Exports the final dataset to a configurable CSV path
- Keeps `OPENAI_API_KEY` outside source code in a `.env` file

## Technology Stack

- **Language:** Python
- **Data processing:** Pandas
- **Environment variables:** python-dotenv
- **AI integration:** OpenAI Python SDK
- **Structured response validation:** Pydantic
- **Input/output format:** CSV
- **License:** MIT

## Repository Structure

```text
ETL_Automation/
├── student_etl.py                  # ETL pipeline and command-line interface
├── student_enrollment_raw.csv      # Example raw student enrollment dataset
├── students_cleaned.csv            # Generated cleaned output
├── .gitignore                      # Python and environment exclusions
├── LICENSE                         # MIT license
└── README.md                       # Project documentation
```

## ETL Workflow

### Step 1: Upload or Select the CSV

The script:

1. Uses the input path supplied as a positional argument, if provided.
2. Otherwise searches the current directory for candidate CSV files.
3. Automatically selects the CSV if exactly one valid candidate is available.
4. Prompts for a file path if multiple or no candidate CSV files are found.
5. Excludes `students_cleaned.csv` and other files ending in `_cleaned.csv`.
6. Raises an error if the supplied path does not exist or is not a CSV file.

### Step 2: Read the Raw File

Pandas reads the CSV with `dtype=str` and `keep_default_na=False`, ensuring that values remain strings and empty fields are represented consistently.

The following fields are expected:

| Column | Purpose |
|---|---|
| `Student_ID` | Unique student identifier |
| `Name` | Full student name |
| `Email` | Student email address |
| `Phone` | Phone number |
| `Course` | Enrolled course |
| `Fee_Paid` | Fee payment status |
| `City` | Student city |
| `Enrolled_Date` | Original enrollment date |

### Step 3: Clean Data with OpenAI

The script processes the records in batches of 25 and sends them to the `gpt-4o-mini` model.

It generates data using two Pydantic models:

- `CleanedStudent`: defines the cleaned structure for one student
- `CleanedStudentBatch`: defines a batch containing multiple cleaned students

The model applies these cleaning rules:

- **Student ID:** Returned as an uppercase value
- **Name:** Extra whitespace is removed and capitalization is corrected
- **Email:** Converted to lowercase
- **Email validity:** Set to `true` only for a valid `user@domain.tld` format
- **Phone:** Contains digits only
- **Indian phone number:** Removes the leading `91` from 12-digit numbers
- **Course:** Cleaned and standardized
- **Fee status:** Interpreted as boolean
  - `true`, `yes`, `paid`, `completed`, and `settled` become `true`
  - `false`, `no`, `unpaid`, `pending`, `due`, and `not paid` become `false`
- **City:** Extra whitespace is removed and capitalization is corrected
- **Date:** Converted to `YYYY-MM-DD` when valid, otherwise set to `null`

### Step 4: Export the Cleaned CSV

The script:

1. Converts Pydantic model output into dictionaries
2. Creates a Pandas DataFrame
3. Removes duplicate records based on `Student_ID`, retaining the first occurrence
4. Writes the result to `students_cleaned.csv` by default
5. Prints the number of exported records and the absolute output path

## Prerequisites

Install the following before running the project:

- Python 3.10 or later
- pip
- Git
- An active OpenAI API key with access to `gpt-4o-mini`
- Billing/API usage enabled for the OpenAI account

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/Karthickbalakrishnan/ETL_Automation.git
cd ETL_Automation
```

### 2. Create a Virtual Environment

**Windows:**

```powershell
py -m venv .venv
.venv\Scripts\activate
```

**macOS/Linux:**

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 3. Install Dependencies

The project does not currently include a `requirements.txt` file. Install the dependencies directly:

```bash
python -m pip install --upgrade pip
python -m pip install pandas python-dotenv openai pydantic
```

### 4. Configure the OpenAI API Key

Create a `.env` file in the project root:

```text
OPENAI_API_KEY=your_openai_api_key_here
```

Alternatively, set the API key in the current terminal session.

**Windows PowerShell:**

```powershell
$env:OPENAI_API_KEY="your_openai_api_key_here"
```

**macOS/Linux:**

```bash
export OPENAI_API_KEY="your_openai_api_key_here"
```

Do not commit the `.env` file or expose the API key in source code.

## Running the Pipeline

### Use the Example File

Because `student_enrollment_raw.csv` is the only candidate CSV in the repository root, run:

```bash
python student_etl.py
```

The script automatically:

- Detects `student_enrollment_raw.csv`
- Sends student records to OpenAI in batches
- Removes duplicate student IDs
- Creates `students_cleaned.csv`

### Specify an Input File

```bash
python student_etl.py path/to/raw_students.csv
```

Example:

```bash
python student_etl.py student_enrollment_raw.csv
```

### Specify an Output File

```bash
python student_etl.py -o cleaned/students.csv
```

If the output directory does not exist, create it first:

```bash
mkdir -p cleaned
python student_etl.py student_enrollment_raw.csv -o cleaned/students.csv
```

### Specify Both Input and Output

```bash
python student_etl.py student_enrollment_raw.csv -o cleaned/students.csv
```

### View Command-Line Help

```bash
python student_etl.py --help
```

The available options are:

```text
positional arguments:
  input_csv             Path to raw input CSV

options:
  -o OUTPUT, --output OUTPUT
                        Path for the cleaned output CSV
                        (default: students_cleaned.csv)
```

## Example Input and Output

### Raw Data

```csv
STU1001,manoj k,manojk@gmail,9876543210,python,yes,Chennai,23-05-2025
```

### Cleaned Data

```csv
STU1001,Manoj K,manojk@gmail,False,9876543210,Python,True,Chennai,
```

This demonstrates several transformations:

- Name changed from `manoj k` to `Manoj K`
- Course changed from `python` to `Python`
- Fee status changed from `yes` to `True`
- Email marked invalid because `manojk@gmail` lacks a top-level domain

## Example Dataset

The repository includes 50 raw records:

- Source: `student_enrollment_raw.csv`
- Output: `students_cleaned.csv`
- Source IDs range from `STU1001` through `STU1050`

The example output adds:

- A `Email_Valid` boolean column
- Boolean `Fee_Paid` values
- Standardized text values
- Standardized `YYYY-MM-DD` dates where possible
- `null` values represented as empty CSV fields

## Using the Pipeline Programmatically

The pipeline can also be called from Python:

```python
from student_etl import run_etl

output_path = run_etl(
    input_csv="student_enrollment_raw.csv",
    output_csv="students_cleaned.csv",
)

print(output_path)
```

Individual stages are available as separate functions:

```python
from student_etl import (
    step_1_upload_csv,
    step_2_read_file,
    step_3_clean_with_openai,
    step_4_export_cleaned_csv,
)

csv_path = step_1_upload_csv("student_enrollment_raw.csv")
records = step_2_read_file(csv_path)
cleaned_records = step_3_clean_with_openai(records)
step_4_export_cleaned_csv(cleaned_records, "students_cleaned.csv")
```

## API and Cost Considerations

Each run sends the raw student records to OpenAI. Costs depend on:

- Number of records
- Batch size
- Model usage and token count
- OpenAI pricing

The default batch size is 25 and the default model is `gpt-4o-mini`.

## Current Limitations

- The script requires an internet connection and a valid OpenAI API key.
- There is no `requirements.txt`, `pyproject.toml`, test suite, or Docker configuration.
- The source CSV structure is not validated before processing.
- Duplicate records are removed only by `Student_ID`.
- Date parsing depends on the model's interpretation rather than an explicit local parser.
- AI cleaning may produce nondeterministic results.
- Raw and cleaned files should not contain sensitive personal data unless required controls are in place.
- The script does not add automatic retry logic or fallback handling for OpenAI request failures.
- The script does not log every API request or create a separate report for rejected/invalid records.

## Troubleshooting

### `OPENAI_API_KEY not found`

Ensure `.env` contains:

```text
OPENAI_API_KEY=your_openai_api_key_here
```

Or export it as an environment variable before running the script.

### `Valid CSV file not found`

Verify that:

- The input path is correct
- The file exists
- The file extension is `.csv`

### Multiple CSV files detected

Pass the intended file explicitly:

```bash
python student_etl.py path/to/raw_students.csv
```

### Model or API request error

Check:

- Internet connectivity
- OpenAI API key validity
- OpenAI account access and billing
- Availability of `gpt-4o-mini`

## Security Recommendations

- Never commit `.env`
- Rotate exposed API keys immediately
- Use environment variables or a secret manager in production
- Encrypt CSV files containing personally identifiable information
- Apply retention and access-control policies for student data
- Review AI-generated output before downstream use
- Add unit and integration tests for cleaning rules

## Contributing

1. Fork the repository.
2. Create a feature branch.
3. Make focused changes.
4. Run the pipeline against a test CSV.
5. Submit a pull request with the input, output, and behavior changes described.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

## Author

**Karthick Balakrishnan**

[GitHub Profile](https://github.com/Karthickbalakrishnan)
