# Sales Semantic Model - Development Specification

## 📋 Project Overview

This document outlines the comprehensive development specification for creating a Power BI semantic model named "Sales" using the PBIP (Power BI Project) file format and TMDL (Tabular Model Definition Language).

### Business Context
The Sales semantic model will provide a unified view of sales performance across Internet and Reseller channels, enabling sales managers, executives, and product managers to analyze total sales performance, growth trends, and product rankings.

## 🎯 Business Requirements

| Requirement ID | Description | User Story | Expected Behavior |
|---------------|-------------|------------|-------------------|
| **SALES-001** | Total Sales Performance Overview | As a **Sales Manager**, I want to see the total sales amount across all channels (Internet + Reseller) so that I can understand our overall revenue performance. | Create a combined [Total Sales Amount] measure that sums the sales from Internet and Reseller with proper relationship handling through shared dimensions. |
| **SALES-002** | Sales Growth Analysis | As a **Executive Leadership**, I want to see year-over-year and month-over-month sales growth percentages so that I can track business progress against targets. | Create time intelligence measures using SAMEPERIODLASTYEAR and PREVIOUSMONTH functions: [Sales YoY Growth %] and [Sales MoM Growth %] with proper Calendar table relationships. |
| **SALES-003** | Top Performing Products | As a **Product Manager**, I want to identify the top 10 products by sales amount and category so that I can focus marketing efforts on high-performing items. | Create [Product Sales Rank] measure using RANKX function and establish proper relationships between fact tables and DimProduct for accurate product-level aggregations. |

## 🗂️ Data Architecture

### Data Source
- **Server**: `dummyserver.database.windows.net`
- **Database**: `AdventureWorksDW`
- **Storage Mode**: Import for all tables

### Selected Tables Analysis

Based on the requirements analysis and schema review, the following tables are required:

#### Fact Tables
1. **FactInternetSales**
   - Purpose: Internet channel sales transactions
   - Key Columns: SalesAmount, OrderQuantity, ProductKey, CustomerKey, OrderDateKey
   - Relationships: Links to Product, Customer, Calendar

2. **FactResellerSales**
   - Purpose: Reseller channel sales transactions
   - Key Columns: SalesAmount, OrderQuantity, ProductKey, ResellerKey, OrderDateKey
   - Relationships: Links to Product, Reseller, Calendar

#### Dimension Tables
1. **Product** (Consolidated from DimProduct + DimProductSubcategory + DimProductCategory)
   - Purpose: Product information with category hierarchy
   - Key Columns: ProductKey, Product Name, Category, Subcategory, Color, List Price
   - Design: Single denormalized table following star schema principles

2. **Customer** (From DimCustomer)
   - Purpose: Customer demographics and attributes
   - Key Columns: CustomerKey, Customer Name, Geography, Demographics
   - Relationships: Links to InternetSales

3. **Calendar** (Reuse existing Calendar.tmdl)
   - Purpose: Time intelligence and date relationships
   - Key Columns: Date, Year, Month, Quarter, Week
   - Relationships: Links to both fact tables via OrderDate

## 📊 Data Model Design

### Star Schema Structure
```
Calendar (Dimension)
    ↓ (1:Many)
InternetSales (Fact) ← (Many:1) → Product (Dimension)
    ↓ (Many:1)
Customer (Dimension)

Calendar (Dimension)
    ↓ (1:Many)
ResellerSales (Fact) ← (Many:1) → Product (Dimension)
```

### Relationships Specification

| Relationship Name | From Table | From Column | To Table | To Column | Cardinality |
|------------------|------------|-------------|----------|-----------|-------------|
| calendar-internetsales | Calendar | Date | InternetSales | OrderDate | 1:Many |
| calendar-resellersales | Calendar | Date | ResellerSales | OrderDate | 1:Many |
| product-internetsales | Product | ProductKey | InternetSales | ProductKey | 1:Many |
| product-resellersales | Product | ProductKey | ResellerSales | ProductKey | 1:Many |
| customer-internetsales | Customer | CustomerKey | InternetSales | CustomerKey | 1:Many |

## 🔢 Measures Specification

### Base Measures (to be created in respective fact tables)

#### InternetSales Table Measures
1. **[Internet Sales Amount]**
   - DAX: `SUM(InternetSales[SalesAmount])`
   - Description: "Total sales amount from internet channel"
   - Format: Currency

2. **[Internet Sales Quantity]**
   - DAX: `SUM(InternetSales[OrderQuantity])`
   - Description: "Total quantity sold through internet channel"
   - Format: Whole Number

#### ResellerSales Table Measures
1. **[Reseller Sales Amount]**
   - DAX: `SUM(ResellerSales[SalesAmount])`
   - Description: "Total sales amount from reseller channel"
   - Format: Currency

2. **[Reseller Sales Quantity]**
   - DAX: `SUM(ResellerSales[OrderQuantity])`
   - Description: "Total quantity sold through reseller channel"
   - Format: Whole Number

### Combined Measures (to be created in InternetSales table)

#### SALES-001: Total Sales Performance
1. **[Total Sales Amount]**
   - DAX: `[Internet Sales Amount] + [Reseller Sales Amount]`
   - Description: "Combined total sales amount across all channels (Internet + Reseller)"
   - Format: Currency

2. **[Total Sales Quantity]**
   - DAX: `[Internet Sales Quantity] + [Reseller Sales Quantity]`
   - Description: "Combined total quantity sold across all channels"
   - Format: Whole Number

#### SALES-002: Sales Growth Analysis
1. **[Total Sales Amount (ly)]**
   - DAX: `CALCULATE([Total Sales Amount], SAMEPERIODLASTYEAR(Calendar[Date]))`
   - Description: "Total sales amount for the same period last year"
   - Format: Currency

2. **[Sales YoY Growth %]**
   - DAX: `DIVIDE([Total Sales Amount] - [Total Sales Amount (ly)], [Total Sales Amount (ly)])`
   - Description: "Year-over-year sales growth percentage"
   - Format: Percentage

3. **[Total Sales Amount (pm)]**
   - DAX: `CALCULATE([Total Sales Amount], PREVIOUSMONTH(Calendar[Date]))`
   - Description: "Total sales amount for the previous month"
   - Format: Currency

4. **[Sales MoM Growth %]**
   - DAX: `DIVIDE([Total Sales Amount] - [Total Sales Amount (pm)], [Total Sales Amount (pm)])`
   - Description: "Month-over-month sales growth percentage"
   - Format: Percentage

#### SALES-003: Top Performing Products
1. **[Product Sales Rank]**
   - DAX: `RANKX(ALL(Product[Product]), [Total Sales Amount], , DESC)`
   - Description: "Ranking of products by total sales amount (descending)"
   - Format: Whole Number

## 🏗️ File Structure

Following PBIP standards, the project will have this structure:

```
/src
    /Sales.SemanticModel
        /definition
            /tables
                internetsales.tmdl
                resellersales.tmdl
                product.tmdl
                customer.tmdl
                calendar.tmdl
            relationships.tmdl
            expressions.tmdl
            model.tmdl
        definition.pbism
    /Sales.Report
        definition.pbir
    Sales.pbip
```

## 🔧 Power Query Transformations

### Product Table Consolidation
The Product table will be created by joining three source tables:
- DimProduct (base table)
- DimProductSubcategory (for subcategory names)
- DimProductCategory (for category names)

### Date Column Creation
Both fact tables will have OrderDate created from OrderDateKey:
- Convert DateKey (YYYYMMDD integer) to proper DateTime format
- Ensure proper relationship with Calendar table

### Selected Columns Strategy
Only essential columns will be imported to minimize model size:

**InternetSales**: SalesOrderNumber, SalesOrderLineNumber, ProductKey, CustomerKey, OrderDateKey, OrderQuantity, SalesAmount
**ResellerSales**: SalesOrderNumber, SalesOrderLineNumber, ProductKey, ResellerKey, OrderDateKey, OrderQuantity, SalesAmount
**Product**: ProductKey, Product Name, Category, Subcategory, Color, List Price
**Customer**: CustomerKey, Customer Name, Geography Key, Demographics
**Calendar**: All columns from existing Calendar.tmdl

## 📋 Parameters Configuration

Two semantic model parameters will be created:
1. **Parameter_Server**: "dummyserver.database.windows.net"
2. **Parameter_Database**: "AdventureWorksDW"

## 🎨 Naming Conventions

### Tables
- **internetsales** (fact table - plural, lowercase)
- **resellersales** (fact table - plural, lowercase)
- **product** (dimension table - singular, lowercase)
- **customer** (dimension table - singular, lowercase)
- **calendar** (dimension table - singular, lowercase)

### Columns
- Business-friendly names without "fact" or "dim" prefixes
- Lowercase with spaces: "sales amount", "order quantity"
- Descriptive names: "product" instead of "product name"

### Measures
- **[measure name]** for base measures
- **[measure name (ly)]** for last year variants
- **[measure name (pm)]** for previous month variants
- **[measure name (ytd)]** for year-to-date variants

## ✅ Quality Assurance

### Validation Steps
1. **Data Validation**: Verify all relationships are active and properly configured
2. **Measure Testing**: Validate all measures return expected results
3. **Performance Testing**: Ensure queries perform efficiently
4. **BPA Validation**: Run Best Practice Analyzer to identify and resolve critical issues

### Success Criteria
- All three business requirements (SALES-001, SALES-002, SALES-003) are fully satisfied
- Star schema design is implemented correctly
- All measures have proper descriptions and formatting
- Model passes BPA validation with no critical errors
- Relationships support proper cross-filtering behavior

## 🚀 Implementation Timeline

1. **Phase 1**: Create PBIP project structure and parameters
2. **Phase 2**: Build dimension tables (Product, Customer, Calendar)
3. **Phase 3**: Create fact tables (InternetSales, ResellerSales)
4. **Phase 4**: Define relationships between tables
5. **Phase 5**: Implement all required measures
6. **Phase 6**: Validate and test the complete model
7. **Phase 7**: Run BPA analysis and resolve any issues

## 📝 Notes

- Calendar table will reuse the existing Calendar.tmdl from `.resources/tmdl/Calendar.tmdl`
- All tables will use Import storage mode for optimal performance
- The model follows star schema principles for simplified querying
- Measures are distributed across appropriate fact tables to avoid duplication
- Server and database connections are parameterized for flexibility
