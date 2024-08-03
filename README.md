# Bizcard-Project

%%writefile app.py

import streamlit as st
import easyocr
from PIL import Image
import pandas as pd
import numpy as np
from streamlit_option_menu import option_menu
import re
import io
import sqlite3

def image_to_text(uploaded_file):
    byte_stream = io.BytesIO(uploaded_file.read())
    input_image = Image.open(byte_stream)
    image_array = np.array(input_image)
    reader = easyocr.Reader(["en"])
    result = reader.readtext(image_array, detail=0)
    return result, input_image

def extracted_text(texts):
    extracted_dict = {
        "Name": [],
        "Designation": [],
        "Phone": [],
        "Website": [],
        "Email": [],
        "Company_Name": [],
        "Address": [],
        "Pincode": []
    }

    if len(texts) > 0:
        extracted_dict["Name"].append(texts[0])

    if len(texts) > 1:
        extracted_dict["Designation"].append(texts[1])

    for text in texts[2:]:
        if isinstance(text, str):
            if text.startswith("+") or (text.replace("-", "").isdigit() and "-" in text):
                extracted_dict["Phone"].append(text)
            elif "@" in text and "." in text:
                extracted_dict["Email"].append(text)
            elif "www" in text.lower():
                extracted_dict["Website"].append(text.lower())
            elif re.match(r'^[A-Za-z]', text):
                extracted_dict["Company_Name"].append(text)
            elif "Tamil Nadu" in text or "TamilNadu" in text or text.isdigit():
                extracted_dict["Pincode"].append(text)
            else:
                text = re.sub(r'[,;]', '', text)
                extracted_dict["Address"].append(text)

    for key, value in extracted_dict.items():
        if len(value) > 0:
            extracted_dict[key] = " ".join(value)
        else:
            extracted_dict[key] = "NA"

    return extracted_dict

# Streamlit Part
st.set_page_config(layout="wide")
st.title("Extracting Data from Business Cards")

select = option_menu("Main menu", ["Home", "Upload & Modifying", "Delete"], icons=['house', 'upload', 'trash'])
if select == "Home":
    st.write("Welcome to the Bizcard Project! Here you can manage your business card data.")

elif select == "Upload & Modifying":
    uploaded_image = st.file_uploader("Upload the Image", type=["png", "jpg", "jpeg"])

    if uploaded_image is not None:
        st.image(uploaded_image, width=100)

        result_image, input_image = image_to_text(uploaded_image)
        text_dict = extracted_text(result_image)

        if text_dict:
            st.success("Text Extracted Successfully")

        df = pd.DataFrame([text_dict])

        # Convert image to bytes
        image_bytes = io.BytesIO()
        input_image.save(image_bytes, format="PNG")
        image_data = image_bytes.getvalue()

        # Store the image on disk and save the path
        image_path = "uploaded_image.png"
        input_image.save(image_path)

        # Update DataFrame to include the image path
        df["Image"] = [image_path]

        st.dataframe(df)

        button_1 = st.button("Save")

        if button_1:
            with sqlite3.connect('bizcard_info.db') as mydb:
                mycursor = mydb.cursor()
                create_query = """CREATE TABLE IF NOT EXISTS Bizcard_info (
                    Name TEXT,
                    Designation TEXT,
                    Phone TEXT,
                    Website TEXT,
                    Email TEXT,
                    Company_Name TEXT,
                    Address TEXT,
                    Pincode Varchar(100),
                    Image TEXT)"""
                mycursor.execute(create_query)
                mydb.commit()

                insert_query = """INSERT INTO Bizcard_info
                                (Name, Designation, Phone, Website, Email, Company_Name, Address, Pincode, Image)
                                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)"""
                data = df.iloc[0].tolist()
                mycursor.execute(insert_query, data)
                mydb.commit()

                st.success("Data Saved Successfully")

        method = st.radio("Select the method", ["None", "Preview", "Modified"])

        if method == "Preview":
            with sqlite3.connect('bizcard_info.db') as mydb:
                mycursor = mydb.cursor()
                select_query = "SELECT * FROM Bizcard_info"
                mycursor.execute(select_query)
                table = mycursor.fetchall()
                table_df = pd.DataFrame(table, columns=["Name","Designation","Phone","Website","Email",
                                                        "Company_Name","Address","Pincode","Image"])
                st.dataframe(table_df)

        elif method == "Modified":
            with sqlite3.connect('bizcard_info.db') as mydb:
                mycursor = mydb.cursor()
                select_query = "SELECT * FROM Bizcard_info"
                mycursor.execute(select_query)
                table = mycursor.fetchall()
                table_df = pd.DataFrame(table, columns=["Name","Designation","Phone","Website","Email",
                                                        "Company_Name","Address","Pincode","Image"])
                st.dataframe(table_df)

                column_1, column_2 = st.columns(2)
                with column_1:
                    selected_name = st.selectbox("Select the Name", table_df["Name"].unique())

                df_3 = table_df[table_df["Name"] == selected_name]

                with column_1:
                    Mod_name = st.text_input("Name", df_3["Name"].values[0])
                    Mod_Desig = st.text_input("Designation", df_3["Designation"].values[0])
                    Mod_Com_Name = st.text_input("Company_Name", df_3["Company_Name"].values[0])
                    Mod_Phone = st.text_input("Phone", df_3["Phone"].values[0])
                    Mod_Email = st.text_input("Email", df_3["Email"].values[0])

                with column_2:
                    Mod_Website = st.text_input("Website", df_3["Website"].values[0])
                    Mod_Address = st.text_input("Address", df_3["Address"].values[0])
                    Mod_Pincode = st.text_input("Pincode", df_3["Pincode"].values[0])
                    Mod_Image = st.text_input("Image", df_3["Image"].values[0])

                df_4 = df_3.copy()
                df_4.loc[df_4.index[0], ["Name", "Designation", "Company_Name", "Phone", "Email",
                                         "Website", "Address", "Pincode", "Image"]] = [Mod_name, Mod_Desig, Mod_Com_Name,
                                                                                      Mod_Phone, Mod_Email, Mod_Website,
                                                                                      Mod_Address, Mod_Pincode, Mod_Image]

                st.dataframe(df_4)

                button_3 = st.button("Modify")

                if button_3:
                    with sqlite3.connect('bizcard_info.db') as mydb:
                        mycursor = mydb.cursor()
                        mycursor.execute(f"DELETE FROM Bizcard_info WHERE Name='{selected_name}'")
                        mydb.commit()

                        insert_query = """INSERT INTO Bizcard_info
                                          (Name, Designation, Phone, Website, Email, Company_Name, Address, Pincode, Image)
                                          VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)"""
                        data = df_4.iloc[0].tolist()
                        mycursor.execute(insert_query, data)
                        mydb.commit()

                        st.success("Data Modified Successfully")

elif select == "Delete":
      mydb = sqlite3.connect('bizcard_info.db')
      mycursor = mydb.cursor()

      column_1,column_2=st.columns(2)
      with column_1:
        select_query= "select Name from bizcard_info"
        mycursor.execute(select_query)
        table_1=mycursor.fetchall()
        mydb.commit()

        Names=[]
        for i in table_1:
          Names.append(i[0])

        selected_name = st.selectbox("Select the Name",Names)

      with column_2:
        select_query= f"select Designation from bizcard_info where Name= '{selected_name}'"
        mycursor.execute(select_query)
        table_2=mycursor.fetchall()
        mydb.commit()

        Designation=[]
        for J in table_2:
          Designation.append(J[0])

        Selected_Designation = st.selectbox("Select the Designation",Designation)

      if selected_name and Selected_Designation:
        column_1,column_2,column_3=st.columns(3)

        with column_1:
          st.write(f"Name: {selected_name}")
          st.write("")
          st.write("")
          st.write("")
          st.write(f"Designation: {Selected_Designation}")
          st.write("")

        with column_2:
          st.write("")
          st.write("")
          st.write("")
          st.write("")
          # st.write("")

          remove=st.button("Remove",use_container_width=True)

          if remove:
            mydb = sqlite3.connect('bizcard_info.db')
            mycursor = mydb.cursor()

            mycursor.execute(f"DELETE FROM Bizcard_info WHERE Name='{selected_name}' AND Designation='{Selected_Designation}'")
            mydb.commit()

            st.warning("Data Deleted")








