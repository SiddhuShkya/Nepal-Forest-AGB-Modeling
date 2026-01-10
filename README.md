# Nepal Forest AGB Modeling

This project aims to estimate **Above Ground Biomass (AGB)** in Nepal's forests by combining field inventory data with satellite imagery. It utilizes a machine learning approach to model forest carbon based on geospatial and ground-truth data.

## Dataset
The project relies on plot-level estimates from Nepal's national forest inventory (2010-2014).

- Dataset posted on 2023-04-07, 11:13 authored by Shiva Khanal, Matthias M Boer
- Dataset source : [Nepal_forest_AGB_SOC_data.csv
](https://figshare.com/articles/dataset/Plot-level_estimates_of_aboveground_biomass_and_soil_organic_carbon_stocks_from_Nepal_s_forest_inventory/21959636?file=38955398)
- Key Variables :
    - `AGB_tha`: Above Ground Biomass (tonnes/hectare)
    - `SOC_tha`: Soil Organic Carbon (tonnes/hectare)
    - `lat`, `lon`: Georeferenced coordinates
    - `plot_id`: Unique identifier for inventory plots

This dataset includes georeferenced plot-level forest carbon estimates for Nepal as a supplement to a data paper submitted to the Scientific Data journal. The dataset is based on field observations from Nepal's national forest inventory from 2010 to 2014 and includes estimates for two major forest carbon pools: aboveground biomass (AGB) and soil organic carbon (SOC) stocks from 2,009 and 1,156 inventory plots, respectively. The forest AGB includes all trees within each sample plot, including standing dead and stumps, while the forest SOC stock includes the organic carbon in the 0-30 cm depth. The organic litter was excluded from the plot-level estimates of SOC due to missing records for many plots. Other analyses have reported a very low contribution of litter to the total carbon stocks in Nepal's forests (<1%).

---

> The map below visualizes the spatial distribution of the sampled forest plots across Nepal, illustrating the coverage of AGB measurements used in this study.

<img src="images/nepal_agb_map.png"
     alt="Astro project template"
     style="border:1px solid white; padding:1px; background:#fff; width: 3000px" />

## Repository Structure

> The project is organized into clear workflows for handling field data and satellite imagery separately before integration.

```text
.
├── 📁 data                         ## Folder containing all the data for the research and project
│   ├── 📁 field-data               ## Folder that contains all the saved field (ground-truth) data
│   ├── 📁 geojson-data             ## Folder that contains geo-location information of each plots
│   └── 📁 satellite-images         ## Folder that contains satellite images (bands) of our plots
├── 📁 docs                         ## Important documents 
├── 📄 .env.example                 ## Environment variables for Google Earth Engine (GEE)
├── 📁 .git                         ## Folder used by the git to manage our project's version control
├── 📄 .gitignore                   ## File to specify files and folder that needs to be ignored by the git
├── 📁 images                       ## Saved images and plots
├── 📁 notebooks                    ## Folder containing all our notebook files 
│   ├── 📁 Field-Data               ## Notebook file for handling field data
│   └── 📁 Satellite-Data           ## Notebook file for handling satellite imagery data
├── 📄 README.md                    ## Project/Repo Description
├── 📄 requirements.txt             ## Project's dependencies
├── 📁 research                     ## Research paper similar to our study
├── 📄 service-account-key.json     ## GEE account key (Your own key here)
└── 📁 venv                         ## Python virtual environment
``` 

---

## Setup & Installation

1. Clone the repository:
```bash
git clone <repo-url>
cd Nepal-Forest-AGB-Modeling
```

2. Install Dependencies:

> Ensure you have Python installed, then run:

```bash
pip install -r requirements.txt
```

3. Create your own GCP project and get your credentials needed for fetching satellite data using GEE.

> Ensure you have your porject-id, service-account and service-account-key. If not go to [GEE-Sevice-Account-Setup](https://github.com/SiddhuShkya/Nepal-Forest-AGB-Modeling/blob/main/docs/Google_Earth_Engine_Service_Account_Setup.pdf) to set one up.

```python
## .env

PROJECT_ID='your-project-id-here'
SERVICE_ACCOUNT='your-service-account-here'
```

```text
.
├── data
├── docs
├── .env
├── .env.example
├── .git
├── .gitignore
├── images
├── notebooks
├── README.md
├── requirements.txt
├── research
├── service-account-key.json <------------ # Your service-account-key.json here
└── venv
```

4.  **Earth Engine Authentication**:
> To use the satellite data fetching notebooks, you must authenticate with Google Earth Engine:
```bash
earthengine authenticate
```

## Usage Workflow
1.  **Prepare Field Data**: Run the notebooks in `notebooks/Field-Data/` to clean and understand your ground truth.
2.  **Fetch Imagery**: Use `notebooks/Satellite-Data/` to download corresponding satellite bands for your plot locations.
3.  **Analysis**: Use `main.ipynb` to merge these datasets and train your AGB prediction models.