# nyhousing
NY Housing data analysis
"""

import pydeck
import streamlit as st
import pandas as pd
import matplotlib.pyplot as plt
import pydeck as pdk


def loadData(file="NY-House-Dataset.csv"):                  

    df = pd.read_csv(file)

    # clean sublocality data into the common 5 borough names
    bronx_list = ["bronx", "east bronx", "riverdale", "bronx country", "the bronx"]
    brooklyn_list = ["brooklyn", "brooklyn heights", "coney island", "dumbo", "fort hamilton", "kings county",
                     "snyder avenue"]
    queens_list = ["queens", "flushing", "jackson heights", "queens county", "rego park"]
    manhattan_list = ["manhattan", "new york", "new york county"]
    staten_list = ["staten island", "richmond county"]

    map_borough = {"Bronx": bronx_list,
                   "Brooklyn": brooklyn_list,
                   "Queens": queens_list,
                   "Manhattan": manhattan_list,
                   "Staten Island": staten_list}

    def standardize(name):
        name = str(name).lower()                            
        for borough, name_list in map_borough.items():
            for i in name_list:                             
                if i in name:
                    return borough
        return None

    # add a borough column with cleaned data
    df['BOROUGH'] = df['SUBLOCALITY'].apply(standardize)

    return df

def price_stats(prices):                                    
    return prices.mean(), prices.median(), prices.min(), prices.max()

def homepage():
    st.title("New York Housing Market Analysis")
    st.write("""
            Welcome to the NY Housing Explorer!\n
            About the Dataset:\n
            - Contains property listings across New York boroughs\n
            - Includes price, bedrooms, bathrooms, square footage, and location data\n
            - Covers various property types (Condos, Houses, Co-ops, etc.\n
            Use the sidebar to navigate through different analyses and visualizations.""")
    st.image("NY.jpg")                                      

# Query 1: Price Distribution by Neighborhood
def priceDistribution(df):
    st.title("Price Distribution by Neighborhood")
    st.write("Analyze housing prices across New York City boroughs")

    # User select a borough
    st.subheader("Select Borough")
    boroughs = sorted(df['BOROUGH'].unique())                   
    selected = st.selectbox("Choose a borough:", boroughs)      
    borough_data = df.query("BOROUGH == @selected")            

    # Calculate and display statistics
    prices = borough_data['PRICE']
    avg, median, min, max = price_stats(prices)                 

    col1, col2 = st.columns([1, 2])

    with col1:
        st.subheader(f"Price Statistics")
        st.write(f"Average Price: ${avg:,.0f}")
        st.write(f"Median Price: ${median:,.0f}")
        st.write(f"Lowest Price: ${min:,.0f}")
        st.write(f"Highest Price: ${max:,.0f}")

    with col2:
        fig, ax = plt.subplots(figsize=(6,3))

        # Filter out extreme outliers
        Q1 = prices.quantile(0.25)
        Q3 = prices.quantile(0.75)
        IQR = Q3 - Q1
        filtered = prices.loc[prices.between(Q1 - 1.5 * IQR, Q3 + 1.5 * IQR)]

        ax.boxplot(filtered, vert=False)                        
        ax.set_title(f"Price Spread in {selected}")
        ax.set_xlabel("Price (Millions)")
        st.pyplot(fig)

# Query 2: Luxury Property Finder
def luxuryFinder(df):
    st.title("Luxury Property Finder")
    st.write("Find properties with more bathrooms than bedrooms, often indicating luxury features like private bathrooms or powder rooms")

    # create two columns for inputs
    col1, col2 = st.columns(2)
    with col1:
        property_type = sorted(df['TYPE'].unique())             
        selected_type = st.selectbox("Property Type:", ["Select a property type"] + list(property_type)) 
    with col2:
        minbeds = st.slider("Minimum Bedrooms:", 1, 10, 2)      

    luxury = (df.query("BEDS >= @minbeds and BATH > BEDS")      
            .query("TYPE == @selected_type or @selected_type == 'Select a property type'")
            .sort_values("BEDS"))                               
    st.write(f"Found {len(luxury)} luxury properties matching your criteria!")

    display_columns = ['TYPE', 'PRICE', 'BEDS', 'BATH', 'PROPERTYSQFT', 'BOROUGH', 'ADDRESS']   
    st.dataframe(luxury[display_columns])

    # Create map with hovering
    st.subheader("Map View")                                    
    map_df = luxury[['LATITUDE', 'LONGITUDE', 'PRICE', 'BEDS', 'BATH', 'ADDRESS']].copy()
    layer = pdk.Layer(
        'ScatterplotLayer',
        data=map_df,
        get_position=['LONGITUDE', 'LATITUDE'],
        get_color=[255, 30, 0, 160],
        get_radius=70,
        pickable=True
    )

    # Customize hover content
    tooltip ={
        "html": """<b>Price:</b> ${PRICE}<br/>
                <b>Beds:</b> {BEDS}<br/>
                <b>Baths:</b> {BATH}<br/>
                <b>Address:</b> {ADDRESS} """,
        "style": {"backgroundColor": "white",
                  "color": "black",
                  "padding": "5px"}
    }

    # Display map
    deck = pdk.Deck(
        layers=[layer],
        initial_view_state=pydeck.ViewState(latitude=40.7128, longitude=-74.0060, zoom=11), # NYC Center
        tooltip=tooltip
    )
    st.pydeck_chart(deck)

# Query 3: Price Heatmap
def priceHeatmap(df):
    st.title("Price Heatmap")
    st.write("See where expensive properties are concentrated across New York City")

    st.subheader("Select area to analyze")
    boroughs = sorted(df['BOROUGH'].unique())                   
    selected = st.selectbox("Choose a borough:", boroughs)      
    heatmap_data = df[df['BOROUGH'] == selected]               

    # Create map
    st.write(f"Showing price concentration in {selected}. Warmer colors = higher price density")
    map_df = heatmap_data[['LATITUDE', 'LONGITUDE', 'PRICE']].copy()    
    heatmap_layer = pdk.Layer(                                 
        'HeatmapLayer',
        data=map_df,
        get_position=['LONGITUDE', 'LATITUDE'],
        get_weight='PRICE',
        color_range=[
            [0, 0, 255, 100],   # Blue - lowest
            [0, 255, 255, 150], # Cyan
            [0, 255, 0, 200],   # Green
            [255, 255, 0, 250], # Yellow
            [255, 165, 0, 250], # Orange
            [255, 0, 0, 250]    #Red - highest
        ]
    )

    # Display map
    tooltip ={
        "html": "<b>Price Density Heatmaps</b><br/>",
        "style": {"backgroundColor": "white",
                  "color": "black"}
    }
    deck = pdk.Deck(
        layers=heatmap_layer,
        initial_view_state=pydeck.ViewState(latitude=40.7128, longitude=-74.0060, zoom=10),  # NYC Center
        tooltip=tooltip
    )
    st.pydeck_chart(deck)

# Query 4: Price vs Property Size
def priceVSsize(df):
    st.title("Price vs Property Size")
    st.write("How does property prices relate to their square footage across NYC boroughs?")

    plot_df = df[['BOROUGH','PRICE', 'PROPERTYSQFT']].copy()    
    boroughs = plot_df['BOROUGH'].unique()

    # Assign colors to each borough
    colors = {
        'Manhattan': 'red',
        'Brooklyn': 'blue',
        'Queens': 'green',
        'Bronx': 'orange',
        'Staten Island': 'purple'
    }

    for borough in boroughs:                                
        col1, col2 = st.columns([3, 1])

        # Create plot
        with col1:
            fig, ax = plt.subplots(figsize=(12, 8))
            color = colors.get(borough)                      

            borough_data = plot_df[plot_df['BOROUGH'] == borough]  

            # Remove outliers and odd data
            prices = borough_data['PRICE'].quantile(0.99)
            sqft = borough_data['PROPERTYSQFT'].quantile(0.99)
            borough_data = borough_data.query(              
                "PROPERTYSQFT != 2184.207862 and PRICE <= @prices and PROPERTYSQFT <= @sqft"
            )

            ax.scatter(                                      
                borough_data['PROPERTYSQFT'],
                borough_data['PRICE'],
                label=borough,
                color=color,
                s=40
                )

            ax.set_title("Price VS Property Size by Borough")
            ax.set_xlabel("Property Size (Sqft")
            ax.set_ylabel("Price (Millions")
            ax.legend()
            st.pyplot(fig)

        # Show average price/sqft
        with col2:
            borough_data['price_sqft'] = borough_data['PRICE'] / borough_data['PROPERTYSQFT']
            avg = borough_data['price_sqft'].mean()
            st.metric("Avg Price/Sqft", f"${avg:,.0f}")

def main():
    df = loadData()

    st.sidebar.title("NY Housing Explorer")                 
    page = st.sidebar.radio("Navigate to:", [
        "Home",
        "Price Distribution by Borough",
        "Luxury Property Finder",
        "Price Heatmap",
        "Price vs Property Size Analysis"
    ])

    if page == "Home":
        homepage()
    if page == "Price Distribution by Borough":
        priceDistribution(df)
    if page == "Luxury Property Finder":
        luxuryFinder(df)
    if page == "Price Heatmap":
        priceHeatmap(df)
    if page == "Price vs Property Size Analysis":
        priceVSsize(df)

if __name__ == "__main__":
    main()
