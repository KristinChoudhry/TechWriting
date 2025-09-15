# Syntax Reference Guide

Creates a new product with specified configuration parameters. Use this when adding catalog items and setting rating methods and attributes.  

## Syntax  

Navigation: Products → Product Management → Products → New  

Required fields:  
- Product Name  

Optional configurations:  
- Override Permissions  
- Rating Method  
- Contract Settings  
- Pricing Attributes  

## Required fields  

| Field         | Type | Required | Description                        |  
|---------------|------|----------|------------------------------------|  
| Product Name  | Text | Yes      | Unique identifier for the product  |  

## Optional configuration fields  

| Field                                   | Type     | Required | Description                                      |  
|-----------------------------------------|----------|----------|--------------------------------------------------|  
| Override Permission                     | Dropdown | No       | Controls ability to modify rating/price          |  
| Product Status                          | Dropdown | No       | Controls availability (set after initial save)   |  
| Contractual Rate Source Required        | Checkbox | No       | Forces contract‑level rates                      |  
| Restrict Use of Unreferenced Contracts  | Checkbox | No       | Limits to contract rates on current account      |  
| Specify Default Rating Method           | Checkbox | No       | Enables rating method configuration              |  

## Override Permission options  

| Option                  | Description                                           | Default |  
|-------------------------|-------------------------------------------------------|---------|  
| No Restriction          | Allow rating method and rate changes                  | Yes     |  
