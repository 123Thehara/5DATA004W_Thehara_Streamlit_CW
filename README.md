import streamlit as st
import plotly.express as px
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt
import os
import warnings

warnings.filterwarnings('ignore')

# Reading CSV files
df_orders = pd.read_csv('orders_cleaned.csv')
df_association_rules = pd.read_csv('association_rules.csv')

# Setting page configuration
st.set_page_config(page_title="Superstore!!!", page_icon=":bar_chart:", layout="wide")

# Title of the dashboard
st.title(" :chart_with_upwards_trend: Minger Electronics Dashboard")
st.markdown('<style>div.block-container{padding-top:1rem;}</style>', unsafe_allow_html=True)

# Splitting the layout into two columns
col1, col2 = st.columns((2))
df_orders["Order Date"] = pd.to_datetime(df_orders["Order Date"])

# Setting start and end dates for date range filter
startDate = pd.to_datetime(df_orders["Order Date"]).min()
endDate = pd.to_datetime(df_orders["Order Date"]).max()

# Date range filter
with col1:
    date1 = pd.to_datetime(st.date_input("Start Date", startDate))

with col2:
    date2 = pd.to_datetime(st.date_input("End Date", endDate))

# Filtering data based on date range
df_orders = df_orders[(df_orders["Order Date"] >= date1) & (df_orders["Order Date"] <= date2)].copy()

# Splitting the layout into two columns for filters
col1, col2 = st.columns((2))

# Sidebar for dashboard filters
st.sidebar.title("Dashboard Filters")

# Tabs for different sections of the dashboard
order_details, association_rules = st.tabs(['Order Details Analysis','Market Basket Analysis'])

# Section for Order Details
with order_details:
    st.header("Order Details Dataset")
    st.write(df_orders)

    # Product Market
    market = st.sidebar.selectbox('Select your Market:', df_orders['Market'].unique())
    # Product Category
    category = st.sidebar.multiselect('Select your Category:', df_orders['Category'].unique())

    # Filtering the dashboard using the Market and Product categories
    if market and category:  
        filtered_data = df_orders[(df_orders["Market"].isin([market])) & (df_orders["Category"].isin(category))]
    elif market:  
        filtered_data = df_orders[df_orders["Market"].isin([market])]
    elif category:  
        # Retrieve sub-categories belonging to the selected category
        subcategories = df_orders[df_orders["Category"].isin(category)]["Sub-Category"].unique().tolist()
        # Filter based on both category and its sub-categories
        filtered_data = df_orders[(df_orders["Category"].isin(category)) | (df_orders["Sub-Category"].isin(subcategories))]
    else:
        filtered_data = df_orders.copy()


    ## Charts for the df_orders dataset ##

    # Sales by category
    st.subheader('Sales by Category')
    grp = filtered_data.groupby(by=['Category'], as_index=False)['Sales'].sum()
    fig1 = px.bar(grp,x='Category', y='Sales', height=600, width=700)
    st.plotly_chart(fig1, use_container_width=True)   


    # Sales by sub category
    st.subheader('Sales by Sub Category')
    grp = filtered_data.groupby(by=['Sub-Category'], as_index=False)['Sales'].sum()
    fig2 = px.bar(grp,x='Sub-Category', y='Sales', height=600, width=700, color_discrete_sequence=['#1f77b4'])  # Steel blue color
    st.plotly_chart(fig2, use_container_width=True) 

    
    # Sales by ship mode
    st.subheader ('Sales by Ship Mode')
    grp = filtered_data.groupby(by=['Ship Mode'], as_index=False)['Sales'].sum()
    fig3 = px.box(filtered_data, x='Ship Mode', y='Sales', height=400, width=600)
    st.plotly_chart(fig3, use_container_width=True)


    # Profit by country
    st.subheader('Profit by Country')
    grp = filtered_data.groupby(by=['Country'], as_index=False)['Profit'].sum()
    fig4 = px.bar(grp, x="Country", y="Profit", color_discrete_sequence=['#87CEEB'])  # Sky blue color
    st.plotly_chart(fig4, use_container_width=True)


    # Profit by ship mode
    st.subheader('Profit by Ship Mode')
    grp = filtered_data.groupby(by=['Ship Mode'], as_index=False)['Profit'].sum()
    fig5 = px.box(filtered_data, x='Ship Mode', y='Profit', height=400, width=600, color_discrete_sequence=['#87CEEB'])  # Sky blue color
    st.plotly_chart(fig5, use_container_width=True)

    # Scatter plot to show the relationship between 'Profit' and 'Sales'
    chart1, chart2 = st.columns((2))
    with chart1:                          
        st.subheader('Profit Distribution by Segment')
        fig6 = px.pie(df_orders, values="Sales", names='Segment', color_discrete_sequence=['#4169E1', '#6495ED ', '#1f77b4'])  # Royal blue, Cornflower blue, Steel blue
        fig6.update_traces(text=df_orders['Segment'], textposition='outside')
        st.plotly_chart(fig6, use_container_width=True)


    # Segment-wise sales distribution
    with chart2:
        scatter = px.scatter(df_orders, x = "Quantity", y = "Profit", size='Sales')
        scatter['layout'].update(title="Relationship between Sales and Profits",
                titlefont = dict(size=20), xaxis = dict(title="Sales", titlefont=dict(size=19)),
                yaxis = dict(title = "Profit", titlefont = dict(size=19)))
        st.plotly_chart(scatter, use_container_width=True)


    # Line chart to show sales over time
    filtered_data["month_year"] = filtered_data["Order Date"].dt.to_period("M")
    st.subheader('Sales over time')
    line = pd.DataFrame(filtered_data.groupby(filtered_data["month_year"].dt.strftime("%Y : %b"))["Sales"].sum()).reset_index()
    line = line.sort_values(by="month_year")
    fig7 = px.line(line, x = "month_year", y="Sales", labels = {"Sales": "Amount"}, height=500, width = 1500, template="gridon")
    st.plotly_chart(fig7, use_container_width=True)


    # Line chart to show profit over time
    filtered_data["month_year"] = filtered_data["Order Date"].dt.to_period("M")
    st.subheader('Profit over time')
    line = pd.DataFrame(filtered_data.groupby(filtered_data["month_year"].dt.strftime("%Y : %b"))["Profit"].sum()).reset_index()
    line = line.sort_values(by="month_year")
    fig7 = px.line(line, x = "month_year", y="Profit", labels = {"Profit": "Amount"}, height=500, width = 1500, template="gridon")
    st.plotly_chart(fig7, use_container_width=True)    

# Section for Market Basket Analysis
with association_rules:
    st.header("Market Basket Analysis: Association Rules")
    st.write(df_association_rules)
    
    # Heatmap
    st.subheader("Heat Map")
    # Create the heatmap based on selected axes
    pivot_data = df_association_rules.pivot_table(index=df_association_rules['antecedents'], columns=df_association_rules['consequents'], values='lift')  

    heatfig, ax = plt.subplots(figsize=(10, 6))
    sns.heatmap(pivot_data, ax=ax, annot=True, cmap="viridis") 
    st.pyplot(heatfig)
        
    with st.container():
        fig8 = px.bar(df_association_rules, x='support', y='antecedents', orientation='h', title='Top Antecedents based on Support')
        st.plotly_chart(fig8, use_container_width=True)
    
        fig9 = px.bar(df_association_rules, x='support', y='consequents', orientation='h', title='Top Consequents based on Support')
        fig9.update_traces(marker_color='#87CEEB')  # Sky blue color
        st.plotly_chart(fig9, use_container_width=True)
