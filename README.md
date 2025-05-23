# Comparative Analysis of LLM and GNN in Numerical Decision-Making: A Long-Context Maintenance Perspective

This repository contains the code and resources for the experiment on classifying driver sex using a Large Language Model (BERT) and a Graph Neural Network (GNN). The primary goal is to compare the performance of these two architectures on a structured dataset with a focus on how they handle and utilize numerical and categorical features for a classification task. This experiment is part of the research paper titled "Comparative Analysis of LLM and GNN in Numerical Decision-Making: A Long-Context Maintenance Perspective."

The core of this experiment is implemented in the Jupyter Notebook: `comparison/GNN_LLM_SexOfDriver_Classification_v2.ipynb`.

## Table of Contents

1. [Overview](#overview)
2. [Dataset](#dataset)
3. [Methodology](#methodology)
   - [Data Preprocessing](#data-preprocessing)
   - [BERT Prompt Construction](#bert-prompt-construction)
   - [Tabular Feature Processing](#tabular-feature-processing)
   - [Graph Construction (for GNN)](#graph-construction-for-gnn)
   - [BERT Model](#bert-model)
   - [GNN Model](#gnn-model)
   - [Training and Evaluation](#training-and-evaluation)
4. [Results](#results)
   - [BERT Performance](#bert-performance)
   - [GNN Performance](#gnn-performance)
   - [Comparative Analysis](#comparative-analysis)
   - [Detailed Sample Analysis](#detailed-sample-analysis)
5. [Conclusion](#conclusion)
6. [Setup and Execution](#setup-and-execution)

## Overview

This project investigates the capabilities of BERT (a transformer-based LLM) and a GraphSAGE GNN model in classifying the sex of a driver based on various accident-related features. The `road-safety.csv` dataset provides a rich source of numerical, categorical, and potentially textual information. The notebook implements data preprocessing, model building, training, and evaluation for both architectures, followed by a comparative analysis of their performance metrics. A key aspect of the BERT model is its augmentation with structured tabular data alongside text-based features. The GNN leverages relationships between data instances by constructing a k-Nearest Neighbors graph.

## Dataset

- **Source**: `data/road-safety.csv`
- **Target Variable**: `SexofDriver`
- **Features**: The dataset contains a variety of features including vehicle type, driver age, engine capacity, vehicle manoeuvre, and other accident-related attributes.

## Methodology

The experiment follows a structured approach:

### Data Preprocessing

1.  **Loading and Inspection**: The dataset is loaded, and an initial inspection is performed to understand its structure, data types, and missing values.
2.  **Target Variable Handling**: Rows with missing `SexofDriver` values are dropped. The target variable is label encoded.
3.  **Data Splitting**: The dataset is split into training (70%), validation (15%), and test (15%) sets, stratified by the target variable. The notebook output indicates 111,762 total samples, with 78,233 training nodes for the GNN after splitting.
4.  **Class Weights**: Class weights are computed for the training set to handle potential class imbalance in the target variable.

### BERT Prompt Construction

Textual descriptions (`text_description`) for the BERT model are engineered as follows (details in notebook cell `364bc23f`):

- A predefined list of columns, `text_cols_for_bert_candidates = ['Vehicle_Type', 'Age_of_Driver', 'Engine_Capacity_(CC)', 'Vehicle_Manoeuvre']`, is considered.
- For each row, the values from these columns (if present in the DataFrame) are converted to strings.
- Missing values (NaNs) within these selected text source columns are filled with the string 'Unknown'.
- These string values are then joined together with spaces to form a single `text_description` string for each data instance. The specific operation is `df['text_description'] = df[text_cols_for_bert].apply(lambda x: ' '.join(x), axis=1)`.
- This `text_description` serves as the primary textual input to the BERT model, which is then tokenized using `BertTokenizer.from_pretrained('bert-base-uncased')`.

### Tabular Feature Processing

These features are used for the GNN and as `additional_features` for the BERT model (details in notebook cell `364bc23f`):

- **Categorical Features**:
  - Identified categorical features (object type or low-cardinality numericals, excluding those used for text description and ID columns) are selected.
  - Missing values in these categorical features are imputed with the mode of each column.
  - A cardinality check is performed (e.g., `nunique() < 50`) to exclude features with too many unique values from One-Hot Encoding.
  - Selected categorical features are One-Hot Encoded using `OneHotEncoder(sparse_output=False, handle_unknown='ignore')`.
- **Numerical Features**:
  - Numerical features (excluding IDs, target, and columns already used for text if they were numeric) are selected.
  - Missing values in these numerical features are imputed with the median of each column.
  - These features are then scaled using `StandardScaler`.
- **Combined Features**: The scaled numerical features and one-hot encoded categorical features are concatenated to form a single feature matrix, `combined_features_for_gnn`. The notebook output shows this results in `x=[111762, 26]`, meaning 26 features for each of the 111,762 samples.

### Graph Construction (for GNN)

The graph for the GNN is constructed as detailed in notebook cell `42258907`:

1.  **Node Representation**: Each instance (111,762 vehicle/driver records) in the dataset is treated as a node in the graph.
2.  **Node Features**: The `combined_features_for_gnn` (26 features per node) serve as the initial node features (`pyg_graph.x`).
3.  **Edge Creation**: A k-Nearest Neighbors (k-NN) graph is constructed to define relationships between nodes.
    - The `kneighbors_graph` function from `sklearn.neighbors` is used.
    - The number of neighbors is set to `n_neighbors_for_graph = 10`.
    - The `metric='cosine'` is used to calculate similarity between the `combined_features_for_gnn` of different nodes.
    - `mode='connectivity'` is used, and `include_self=False` ensures no self-loops from this step.
    - The resulting adjacency matrix represents a directed graph.
4.  **Graph Conversion**:
    - The sparse adjacency matrix from `kneighbors_graph` is converted into a `NetworkX` directed graph.
    - This directed graph is then converted to an undirected graph (`G_knn.to_undirected()`). The output shows the graph has 111,762 nodes and 764,849 edges after this step (before conversion to PyG format which might result in `edge_index=[2, 1529698]` due to undirected representation).
    - Finally, the undirected `NetworkX` graph is converted into a PyTorch Geometric `Data` object (`pyg_graph = from_networkx(G_undirected)`).
5.  **Memory Management**: Explicit garbage collection (`gc.collect()`) is called at various stages of graph construction.
6.  **Graph Data**: The final PyTorch Geometric `Data` object (`pyg_graph`) stores node features (`x`), edge connectivity (`edge_index=[2, 1529698]`), labels (`y`), and train/validation/test masks. It has `num_nodes=111762`.

### BERT Model

1.  **Architecture**: `BertClassifierWithAdditionalFeatures`
    - Utilizes a pre-trained `bert-base-uncased` model.
    - The BERT model's `pooler_output` is concatenated with the `additional_features` (`combined_features_for_gnn`).
    - A linear classification layer is added on top.
    - Dropout (`nn.Dropout(0.1)`) is applied.
2.  **Dataset Handling**: `CustomBERTDataset` and `DataLoader`.
3.  **Optimizer**: Adam (`lr=2e-5`).
4.  **Loss Function**: `CrossEntropyLoss` with class weights.

### GNN Model

1.  **Architecture**: `GNNModel`
    - Two `SAGEConv` layers with ReLU and Dropout (`p=0.5`).
    - Output layer with `log_softmax`.
2.  **Input**: PyTorch Geometric `Data` object (`pyg_graph`).
3.  **Optimizer**: Adam (`lr=0.01`, `weight_decay=5e-4`).
4.  **Loss Function**: `NLLLoss` with class weights.

### Training and Evaluation

1.  **Training Loop**:
    - Both models are trained for a specified number of epochs (BERT: 3 epochs, GNN: 50 epochs, as per notebook cell `f0134253` and `76d6ba41`).
    - During training, performance (accuracy and loss) is monitored on the training set.
    - After each epoch (for BERT) or periodically (e.g., every 10 epochs for GNN), the model is evaluated on the validation set using accuracy, loss, precision, recall, and F1-score.
    - BERT training output (Epoch 3): Accuracy: 0.7168, Loss: 0.5467. Validation: Acc: 0.7222, Loss: 0.5449, P: 0.7243, R: 0.7222, F1: 0.7215.
    - GNN training output (Epoch 50): Train Acc: 0.7088, Loss: 0.5705. Val Acc: 0.7105, Loss: 0.5669, F1: 0.7104.
2.  **Metrics**: The primary evaluation metrics collected for both models on the test set are:
    - Accuracy
    - Precision (weighted average)
    - Recall (weighted average)
    - F1-score (weighted average)
    - Confusion Matrix
      Additionally, training and validation loss and accuracy are tracked per epoch.
3.  **Visualization**:
    - Training and validation loss curves over epochs.
    - Training and validation accuracy curves over epochs.
    - Confusion matrices for the test set, showing class-wise performance.
    - A bar chart comparing the final test metrics (Accuracy, Precision, Recall, F1-score) of BERT and GNN.
4.  **Model Saving**: The trained weights of both models are saved to disk (`models/bert_classifier_sexofdriver.pth` and `models/gnn_model_sexofdriver.pth`).

## Results

The performance of both models is evaluated on the held-out test set. The key metrics are presented in the notebook and saved as plots in the `metrics/` directory.

### BERT Performance

- **Test Metrics (from cell output `--- Comparative Metrics ---`)**:
  - Accuracy: 0.7216
  - Precision: 0.7237
  - Recall: 0.7216
  - F1-score: 0.7209
- **Plots**:
  - `metrics/bert_sexofdriver_loss.png`: Training and validation loss over epochs.
  - `metrics/bert_sexofdriver_accuracy.png`: Training and validation accuracy over epochs.
  - `metrics/bert_sexofdriver_cm.png`: Confusion matrix on the test set.

### GNN Performance

- **Test Metrics (from cell output `--- Comparative Metrics ---`)**:
  - Accuracy: 0.7071
  - Precision: 0.7075
  - Recall: 0.7071
  - F1-score: 0.7070
- **Plots**:
  - `metrics/gnn_sexofdriver_loss.png`: Training and validation loss over epochs.
  - `metrics/gnn_sexofdriver_accuracy.png`: Training and validation accuracy over epochs.
  - `metrics/gnn_sexofdriver_cm.png`: Confusion matrix on the test set.

### Comparative Analysis

- A direct comparison of the test metrics (Accuracy, Precision, Recall, F1-score) for BERT and GNN is provided in the notebook (cell `b726a83d` output):
  - **BERT**: Accuracy: 0.7216, Precision: 0.7237, Recall: 0.7216, F1: 0.7209
  - **GNN**: Accuracy: 0.7071, Precision: 0.7075, Recall: 0.7071, F1: 0.7070
- **Plot**: `metrics/comparison_sexofdriver_summary.png`: Bar chart summarizing the performance of both models on the key test metrics.

### Detailed Sample Analysis

The notebook (cell `8a52b1cf`) includes a section for "Detailed Analysis and Sample Inference." For a selected sample (Original DataFrame Index: 26049):

- **Original Data Sample (Index 26049)**:
  - `Vehicle_Type`: 9
  - `Age_of_Driver`: 49
  - `Engine_Capacity_(CC)`: 1461.0
  - `Vehicle_Manoeuvre`: 18
  - `SexofDriver`: 0 (True Label)
  - `Vehicle_Location-Restricted_Lane`: 0
  - `Hit_Object_in_Carriageway`: 0
  - `Hit_Object_off_Carriageway`: 0
- **Text Description (Input to BERT)**: "9 49 1461.0 18"
- **True Label**: 0 (Encoded: 0)

- **BERT Model Inference (Sample 26049)**:

  - Additional Features (first 5): `tensor([[-0.0938, -0.1948, -0.2731, -0.0438,  0.6342]])`
  - Predicted Label: 1 (Encoded: 1)
  - Prediction Probabilities: Class 0: 0.4764, Class 1: 0.5236
  - Confidence in Prediction (Class 1): 0.5236
  - Correct: False

- **GNN Model Inference (Sample 26049)**:
  - Node Features (first 5): `tensor([-0.0938, -0.1948, -0.2731, -0.0438,  0.6342])`
  - Predicted Label: 0 (Encoded: 0)
  - Prediction Probabilities: Class 0: 0.5118, Class 1: 0.4882
  - Confidence in Prediction (Class 0): 0.5118
  - Correct: True

This sample-level analysis shows an instance where the GNN correctly predicted the driver's sex while BERT did not, despite BERT having slightly higher overall test metrics.

## Conclusion

The notebook provides a framework for comparing a BERT-based LLM (augmented with tabular data) and a GNN for a classification task on structured data. Based on the test metrics, BERT (Accuracy: 0.7216) slightly outperformed the GNN (Accuracy: 0.7071) on this specific `SexofDriver` classification task.

- **Performance**: BERT showed slightly better aggregate performance on the test set.
- **Potential Reasons**: Differences could arise from how each model leverages features: BERT processes a textual combination alongside tabular data, while GNN explicitly models relationships based on feature similarity. The chosen k-NN graph structure and GNN architecture (GraphSAGE) influence GNN's performance.
- **Limitations**: The choice of `k=10` for k-NN, specific BERT/GNN architectures, and the extent of hyperparameter tuning are limitations. The textual prompt for BERT is relatively simple.
- **Future Work**: Exploring different graph construction methods, more complex GNN architectures (e.g., GAT), richer textual prompts for BERT, alternative ways to integrate textual and tabular data, and more extensive hyperparameter optimization could yield further insights.

The detailed sample analysis highlights that aggregate metrics don't tell the whole story, as models may perform differently on individual instances.

## Setup and Execution

1.  **Environment**:
    - Ensure Python 3.x is installed.
    - It is highly recommended to use a virtual environment (e.g., `venv` or `conda`).
2.  **Dependencies**: Install the required Python packages. A `requirements.txt` file is usually provided for this. If not, install based on notebook imports:
    ```bash
    pip install pandas numpy torch torchvision torchaudio torch_geometric transformers scikit-learn matplotlib seaborn networkx
    ```
    - Ensure `torch` is installed with the correct CUDA version if GPU support is desired and available. Check PyTorch official website for installation commands.
    - `torch_geometric` installation might require specific PyTorch versions. Refer to its documentation.
3.  **Data**:
    - Place the `road-safety.csv` dataset in the `data/` directory relative to the notebook's location (i.e., `comparison/data/road-safety.csv`).
4.  **Cache**: The BERT model and tokenizer will be downloaded by the `transformers` library, typically to a cache directory (e.g., `cache/` as specified in the notebook, or `~/.cache/huggingface/transformers/`).
5.  **Running the Notebook**:
    - Open and run the `comparison/GNN_LLM_SexOfDriver_Classification_v2.ipynb` notebook in a Jupyter environment (e.g., Jupyter Lab, VS Code with Jupyter extension).
    - The notebook will create `metrics/` and `models/` directories (if they don't exist) within the `comparison/` folder to store output plots and saved model weights, respectively.

This README provides a comprehensive overview of the methodology, results, and conclusions derived from the `GNN_LLM_SexOfDriver_Classification_v2.ipynb` notebook, aligning with the objectives of the research paper.
