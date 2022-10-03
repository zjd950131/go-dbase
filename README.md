# go-dbase - Microsoft Visual FoxPro DBF 

[![GoDoc](https://godoc.org/github.com/golang/gddo?status.svg)](http://godoc.org/github.com/Valentin-Kaiser/go-dbase)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![golangci-lint](https://github.com/Valentin-Kaiser/go-dbase/workflows/golangci-lint/badge.svg)](https://github.com/Valentin-Kaiser/go-dbase)
[![CodeQL](https://github.com/Valentin-Kaiser/go-dbase/workflows/CodeQL/badge.svg)](https://github.com/Valentin-Kaiser/go-dbase)
[![goreport](https://goreportcard.com/badge/github.com/Valentin-Kaiser/go-dbase)](https://goreportcard.com/report/github.com/Valentin-Kaiser/go-dbase)

Golang package for reading FoxPro dBase database files.
This package provides a reader for reading FoxPro database files.

Since these files are almost always used on Windows platforms the default encoding is from Windows-1250 to UTF8 but a universal encoder will be provided for other code pages.
# Features 

There are several similar packages like the [go-foxpro-dbf](https://github.com/SebastiaanKlippert/go-foxpro-dbf) package but they are not suited for our use case, this package implemented:

* Support for FPT (memo) files
* Full support for Windows-1250 encoding to UTF8
* File readers for scanning files (instead of reading the entire file to memory)
* Conversion from and to map, json and struct
* Non blocking IO operation with syscall under windows
* Writing to dBase and memo files

The focus is on performance while also trying to keep the code readable and easy to use.

# Supported column types

At this moment not all FoxPro column types are supported.
When reading column values, the value returned by this package is always `interface{}`. 
If you need to cast this to the correct value helper functions are provided.

The supported column types with their return Go types are: 

| Column Type | Column Type Name | Golang type |
|------------|-----------------|-------------|
| B | Double | float64 |
| C | Character | string |
| D | Date | time.Time |
| F | Float | float64 |
| I | Integer | int32 |
| L | Logical | bool |
| M | Memo  | string |
| M | Memo (Binary) | []byte |
| N | Numeric (0 decimals) | int64 |
| N | Numeric (with decimals) | float64 |
| T | DateTime | time.Time |
| Y | Currency | float64 |

If you need more information about dbase data types take a look here: [Microsoft Visual Studio Foxpro](https://learn.microsoft.com/en-us/previous-versions/visualstudio/foxpro/74zkxe2k(v=vs.80))

# Installation
``` 
go get github.com/Valentin-Kaiser/go-dbase/dbase
```

# Examples

<details>
  <summary>Read</summary>
  
```go
package main

import (
	"fmt"
	"time"

	"github.com/Valentin-Kaiser/go-dbase/dbase"
)

type Test struct {
	ID          int32     `json:"ID"`
	Niveau      int32     `json:"NIVEAU"`
	Date        time.Time `json:"DATUM"`
	TIJD        string    `json:"TIJD"`
	SOORT       float64   `json:"SOORT"`
	IDNR        int32     `json:"ID_NR"`
	UserNR      int32     `json:"USERNR"`
	CompanyName string    `json:"COMP_NAME"`
	CompanyOS   string    `json:"COMP_OS"`
	Melding     string    `json:"MELDING"`
	Number      float64   `json:"NUMBER"`
	Float       int64     `json:"FLOAT"`
	Bool        bool      `json:"BOOL"`
}

func main() {
	// Open the example database file.
	dbf, err := dbase.Open("../test_data/TEST.DBF", new(dbase.Win1250Converter))
	if err != nil {
		panic(err)
	}
	defer dbf.Close()

	// Read the first row (rowPointer start at the first row).
	row, err := dbf.Row()
	if err != nil {
		panic(err)
	}

	// Get the company name field by column name.
	field, err := row.Field(dbf.ColumnPosByName("COMP_NAME"))
	if err != nil {
		panic(err)
	}

	// Change the field value
	field.SetValue("CHANGED_COMPANY_NAME")

	// Apply the changed field value to the row.
	err = row.ChangeField(field)
	if err != nil {
		panic(err)
	}

	// Change a memo field value.
	field, err = row.Field(dbf.ColumnPosByName("MELDING"))
	if err != nil {
		panic(err)
	}

	// Change the field value
	field.SetValue("MEMO_TEST_VALUE")

	// Apply the changed field value to the row.
	err = row.ChangeField(field)
	if err != nil {
		panic(err)
	}

	// Write the changed row to the database file.
	err = row.Write()
	if err != nil {
		panic(err)
	}

	// Create a new row with the same structure as the database file.
	t := Test{
		ID:          99,
		Niveau:      100,
		Date:        time.Now(),
		TIJD:        "00:00",
		SOORT:       101.23,
		IDNR:        102,
		UserNR:      103,
		CompanyName: "NEW_COMPANY_NAME",
		CompanyOS:   "NEW_COMPANY_OS",
		Melding:     "NEW_MEMO_TEST_VALUE",
		Number:      104,
		Float:       105,
		Bool:        true,
	}

	row, err = dbf.RowFromStruct(t)
	if err != nil {
		panic(err)
	}

	// Add the new row to the database file.
	err = row.Write()
	if err != nil {
		panic(err)
	}

	// Print all rows.
	for !dbf.EOF() {
		row, err := dbf.Row()
		if err != nil {
			panic(err)
		}

		// Increment the row pointer.
		dbf.Skip(1)

		// Skip deleted rows.
		if row.Deleted {
			fmt.Printf("Deleted row at position: %v \n", row.Position)
			continue
		}

		// Get the first field by column position
		fmt.Println(row.Values()...)
	}
}
```
</details>

<details>
  <summary>Write</summary>

```go
package main

import (
	"fmt"
	"time"

	"github.com/Valentin-Kaiser/go-dbase/dbase"
)

type Test struct {
	ID          int32     `json:"ID"`
	Niveau      int32     `json:"NIVEAU"`
	Date        time.Time `json:"DATUM"`
	TIJD        string    `json:"TIJD"`
	SOORT       float64   `json:"SOORT"`
	IDNR        int32     `json:"ID_NR"`
	UserNR      int32     `json:"USERNR"`
	CompanyName string    `json:"COMP_NAME"`
	CompanyOS   string    `json:"COMP_OS"`
	Melding     string    `json:"MELDING"`
	Number      float64   `json:"NUMBER"`
	Float       int64     `json:"FLOAT"`
	Bool        bool      `json:"BOOL"`
}

func main() {
	// Open the example database file.
	dbf, err := dbase.Open("../test_data/TEST.DBF", new(dbase.Win1250Converter))
	if err != nil {
		panic(err)
	}
	defer dbf.Close()

	// Read the first row (rowPointer start at the first row).
	row, err := dbf.Row()
	if err != nil {
		panic(err)
	}

	// Get the company name field by column name.
	field, err := row.Field(dbf.ColumnPosByName("COMP_NAME"))
	if err != nil {
		panic(err)
	}

	// Change the field value
	field.SetValue("CHANGED_COMPANY_NAME")

	// Apply the changed field value to the row.
	err = row.ChangeField(field)
	if err != nil {
		panic(err)
	}

	// Change a memo field value.
	field, err = row.Field(dbf.ColumnPosByName("MELDING"))
	if err != nil {
		panic(err)
	}

	// Change the field value
	field.SetValue("MEMO_TEST_VALUE")

	// Apply the changed field value to the row.
	err = row.ChangeField(field)
	if err != nil {
		panic(err)
	}

	// Write the changed row to the database file.
	err = row.Write()
	if err != nil {
		panic(err)
	}

	// Create a new row with the same structure as the database file.
	t := Test{
		ID:          99,
		Niveau:      100,
		Date:        time.Now(),
		TIJD:        "00:00",
		SOORT:       101.23,
		IDNR:        102,
		UserNR:      103,
		CompanyName: "NEW_COMPANY_NAME",
		CompanyOS:   "NEW_COMPANY_OS",
		Melding:     "NEW_MEMO_TEST_VALUE",
		Number:      104,
		Float:       105,
		Bool:        true,
	}

	row, err = dbf.RowFromStruct(t)
	if err != nil {
		panic(err)
	}

	// Add the new row to the database file.
	err = row.Write()
	if err != nil {
		panic(err)
	}

	// Print all rows.
	for !dbf.EOF() {
		row, err := dbf.Row()
		if err != nil {
			panic(err)
		}

		// Increment the row pointer.
		dbf.Skip(1)

		// Skip deleted rows.
		if row.Deleted {
			fmt.Printf("Deleted row at position: %v \n", row.Position)
			continue
		}

		// Get the first field by column position
		fmt.Println(row.Values()...)
	}
}
```
</details>