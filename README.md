# intel-task1
1)contract_analyzer_app.py focuses on the Streamlit interface and user interaction.


import streamlit as st
from contract_analyzer_utils import analyze_contract, detect_deviations


def main():
    st.set_page_config(page_title="Business Contract Analyzer", layout="wide")

    st.title("Business Contract Analyzer")
    st.markdown("""
    This app analyzes business contracts, identifies clauses and subclauses, and detects deviations from a standard template.
    """)

    # Template contract
    template_contract = """
    1. Services. Service Provider agrees to provide and Buyer agrees to purchase the following services for the specific projects described below:
    2. Purchase Price. Buyer will pay to Service Provider and for all obligations specified in this Agreement, if any, as the full and complete purchase price, the sum of ___________.
    3. Payment. Payment for the Services will be by __________, according to the following schedule:
    4. Delivery. Seller shall ship the Goods to Buyer on or before __________ at the following address:
    5. Risk of Loss. Title to and risk of loss of the Goods shall pass to Buyer upon delivery of the Goods to Buyer in accordance with this Agreement.
    6. Security Interest. Buyer hereby grants to Service Provider a security interest in any final products resulting from said services, until Buyer has paid Service Provider in full. Buyer shall sign and deliver any document needed to perfect the security interest that Service Provider reasonably requests.
    """

    # Analyze the template
    template_analysis = analyze_contract(template_contract)

    # File uploader for the actual contract
    uploaded_file = st.file_uploader("Choose a contract file", type="txt")

    if uploaded_file is not None:
        contract_text = uploaded_file.getvalue().decode("utf-8")

        col1, col2 = st.columns(2)

        with col1:
            st.subheader("Uploaded Contract")
            st.text_area("Contract Text", contract_text, height=300)

        # Analyze the uploaded contract
        analysis = analyze_contract(contract_text)

        with col2:
            st.subheader("Contract Analysis")
            for clause, subclauses in analysis.items():
                with st.expander(clause):
                    for subclause in subclauses:
                        st.write(f"- {subclause}")

        # Detect deviations
        deviations = detect_deviations(analysis, template_analysis)

        # Display deviations
        st.subheader("Deviations from Template")
        if deviations:
            for clause, deviation in deviations.items():
                color = ""
                if deviation["status"] == "Missing clause":
                    color = "🔴"
                elif deviation["status"] == "Modified":
                    color = "🟠"
                elif deviation["status"] == "New clause":
                    color = "🟢"

                with st.expander(f"{color} {clause} - {deviation['status']}"):
                    for detail in deviation["details"]:
                        if detail.startswith('- '):
                            st.markdown(f"<font color='red'>{detail}</font>", unsafe_allow_html=True)
                        elif detail.startswith('+ '):
                            st.markdown(f"<font color='green'>{detail}</font>", unsafe_allow_html=True)
                        else:
                            st.write(detail)
        else:
            st.success("No significant deviations detected.")

        # Summary statistics
        st.subheader("Summary Statistics")
        total_clauses = len(analysis)
        total_subclauses = sum(len(subclauses) for subclauses in analysis.values())
        total_deviations = len(deviations)

        col1, col2, col3 = st.columns(3)
        col1.metric("Total Clauses", total_clauses)
        col2.metric("Total Subclauses", total_subclauses)
        col3.metric("Total Deviations", total_deviations)


if __name__ == "__main__":
    main()

   2) contract_analyzer_utils.py contains all the logic for analyzing contracts and detecting deviations.

import re
import nltk
from nltk.tokenize import sent_tokenize
import difflib

# Download necessary NLTK data
nltk.download('punkt', quiet=True)


def extract_clauses(text):
    clause_pattern = r'(\d+\.)\s+([^\n]+)'
    clauses = re.findall(clause_pattern, text)
    return {num.strip('.'): title.strip() for num, title in clauses}


def extract_subclauses(text, clause_num):
    clause_pattern = f'{clause_num}\\.(.*?)(?=\\d+\\.|$)'
    clause_text = re.search(clause_pattern, text, re.DOTALL)
    if not clause_text:
        return []

    sentences = sent_tokenize(clause_text.group(1))
    return [sent.strip() for sent in sentences if sent.strip()]


def analyze_contract(contract_text):
    clauses = extract_clauses(contract_text)

    analysis = {}
    for num, title in clauses.items():
        subclauses = extract_subclauses(contract_text, num)
        analysis[f"{num}. {title}"] = subclauses

    return analysis


def detect_deviations(analysis, template_analysis):
    deviations = {}
    for clause in template_analysis:
        if clause not in analysis:
            deviations[clause] = {"status": "Missing clause", "details": []}
        else:
            template_subclauses = template_analysis[clause]
            actual_subclauses = analysis[clause]

            diff = difflib.ndiff(template_subclauses, actual_subclauses)
            changes = [line for line in diff if line.startswith('+ ') or line.startswith('- ')]

            if changes:
                deviations[clause] = {"status": "Modified", "details": changes}

    for clause in analysis:
        if clause not in template_analysis:
            deviations[clause] = {"status": "New clause", "details": analysis[clause]}

    return deviations
