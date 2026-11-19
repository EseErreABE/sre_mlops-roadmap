# 02 · Frameworks de ML: scikit-learn, XGBoost/LightGBM, PyTorch, TensorFlow

> Naturaleza: **construcción de modelos**. Producen el artefacto que el resto del stack versiona, sirve y monitorea.
> [← Índice](00-indice-roadmap.md) · Anterior: [01](01-lenguajes.md) · Siguiente: [03 Datos](03-procesamiento-datos.md)

---

## Fundamentos que aplican a todos
- Split train/validación/test (temporal si hay series; estratificado si hay desbalance).
- Métricas: clasificación (AUC, F1, precision/recall, log-loss), regresión (RMSE, MAE, MAPE), ranking (NDCG).
- Validación cruzada, tuning de hiperparámetros, *data leakage*, sesgo/varianza, calibración, interpretabilidad (SHAP).
- Reproducibilidad: semillas, versiones fijadas, datos versionados ([08](08-versionado-tracking.md)).

---

## scikit-learn

**Para qué sirve:** ML clásico sobre datos tabulares (preprocesamiento, modelos, evaluación) con API uniforme `fit/predict`.

**Escenario de uso:** baselines, modelos de churn/propensión, clustering de clientes, pipelines de preprocesamiento reutilizables en inferencia.

**Comandos típicos**
```python
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split, cross_val_score, GridSearchCV
from sklearn.metrics import classification_report, roc_auc_score
import joblib

pre = ColumnTransformer([
    ("num", StandardScaler(), num_cols),
    ("cat", OneHotEncoder(handle_unknown="ignore"), cat_cols),
])
pipe = Pipeline([("pre", pre), ("clf", LogisticRegression(max_iter=1000))])
X_tr, X_te, y_tr, y_te = train_test_split(X, y, stratify=y, test_size=0.2, random_state=42)
gs = GridSearchCV(pipe, {"clf__C": [0.1, 1, 10]}, cv=5, scoring="roc_auc").fit(X_tr, y_tr)
print(roc_auc_score(y_te, gs.predict_proba(X_te)[:, 1]))
joblib.dump(gs.best_estimator_, "model.joblib")   # pipeline completo = sin skew en inferencia
```

**Variantes:** `skops` (serialización segura), `imbalanced-learn`, `sklearn-onnx` (exportar a ONNX), Optuna para tuning.

**Requisitos:** Python + NumPy/Pandas; estadística básica, métricas, validación cruzada.

**Conexión:** MLflow `mlflow.sklearn.autolog()` ([08](08-versionado-tracking.md)), BentoML/FastAPI ([10](10-serving.md)), ONNX ([13](13-optimizacion-llm-agentes.md)), Pandera para validar entrada ([09](09-feature-store-calidad.md)).

**Aprende más:** scikit-learn.org/stable/user_guide · *Hands-On ML* (A. Géron) · Kaggle Learn.

---

## XGBoost / LightGBM (y CatBoost)

**Para qué sirve:** gradient boosting de árboles; estado del arte en datos tabulares.

**Escenario de uso:** scoring de riesgo, propensión de compra, feature importance (p. ej. `mejor_agente_b2b_feature_importance`), modelos con muchas variables categóricas.

**Comandos típicos**
```python
import xgboost as xgb, lightgbm as lgb

m = xgb.XGBClassifier(n_estimators=500, max_depth=6, learning_rate=0.05,
                      early_stopping_rounds=50, eval_metric="auc", tree_method="hist")
m.fit(X_tr, y_tr, eval_set=[(X_val, y_val)], verbose=100)
m.save_model("model.json")

l = lgb.LGBMClassifier(num_leaves=63, n_estimators=1000)
l.fit(X_tr, y_tr, eval_set=[(X_val, y_val)], callbacks=[lgb.early_stopping(50)])

import shap
shap.TreeExplainer(m).shap_values(X_te)   # interpretabilidad
```

**Variantes:** XGBoost (robusto, GPU con `device="cuda"`), LightGBM (más rápido en datasets grandes), CatBoost (categóricas nativas). Integración con Spark (`xgboost.spark`).

**Requisitos:** entender boosting, regularización, early stopping, desbalance (`scale_pos_weight`).

**Conexión:** MLflow autolog, SageMaker built-in XGBoost ([14](14-cloud-aws-azure-gcp.md)), OCI Data Science ([15](15-cloud-oci.md)), Triton FIL backend ([10](10-serving.md)), Evidently para drift ([11](11-monitoreo-observabilidad.md)).

**Aprende más:** xgboost.readthedocs.io · lightgbm.readthedocs.io · curso "Gradient Boosting" de StatQuest (YouTube).

---

## PyTorch

**Para qué sirve:** deep learning flexible (visión, NLP, audio, LLM); estándar de facto en investigación y GenAI.

**Escenario de uso:** fine-tuning de modelos de lenguaje o speech-to-text, embeddings, clasificación de texto/audio, modelos personalizados.

**Comandos típicos**
```python
import torch, torch.nn as nn
device = "cuda" if torch.cuda.is_available() else "cpu"
model = MiRed().to(device)
opt = torch.optim.AdamW(model.parameters(), lr=3e-4)
for xb, yb in loader:
    xb, yb = xb.to(device), yb.to(device)
    loss = nn.functional.cross_entropy(model(xb), yb)
    opt.zero_grad(); loss.backward(); opt.step()
torch.save(model.state_dict(), "model.pt")
model.eval()
with torch.inference_mode(): preds = model(x)

torch.compile(model)                                      # aceleración 2.x
torch.onnx.export(model, x, "model.onnx", dynamo=True)    # export a ONNX
```
```bash
nvidia-smi                      # uso GPU
torchrun --nproc_per_node=4 train.py   # entrenamiento distribuido (DDP)
```

**Variantes:** PyTorch Lightning (estructura), Hugging Face Transformers/PEFT (LLM), TorchScript (legado), `torch.compile`, DeepSpeed/FSDP (distribuido).

**Requisitos:** GPU NVIDIA + CUDA compatible para entrenar; álgebra lineal, backprop, optimizadores, manejo de memoria GPU.

**Conexión:** TorchServe/Triton/KServe ([10](10-serving.md)), ONNX/TensorRT/cuantización ([13](13-optimizacion-llm-agentes.md)), W&B/MLflow ([08](08-versionado-tracking.md)), Kubeflow para entrenamiento distribuido ([07](07-orquestacion-pipelines.md)).

**Aprende más:** pytorch.org/tutorials · *Deep Learning with PyTorch* (Manning) · Hugging Face Course · fast.ai.

---

## TensorFlow / Keras

**Para qué sirve:** deep learning con ecosistema de producción maduro (TF Serving, TFLite, TFX).

**Escenario de uso:** modelos heredados, despliegue en móvil/edge (TFLite), pipelines TFX.

**Comandos típicos**
```python
import keras
model = keras.Sequential([keras.layers.Dense(64, activation="relu"), keras.layers.Dense(1, activation="sigmoid")])
model.compile(optimizer="adam", loss="binary_crossentropy", metrics=["AUC"])
model.fit(X, y, validation_split=0.2, epochs=20, callbacks=[keras.callbacks.EarlyStopping(patience=3)])
model.export("saved_model/1")      # formato SavedModel para TF Serving
```

**Variantes:** Keras 3 (multi-backend: TF/JAX/PyTorch), TFLite, TF Serving, TFX, JAX.

**Requisitos:** igual que PyTorch.

**Conexión:** TF Serving/Triton/KServe ([10](10-serving.md)), TensorBoard, Vertex AI ([14](14-cloud-aws-azure-gcp.md)).

**Aprende más:** tensorflow.org/tutorials · keras.io/guides.

---

## Hilando soluciones
**Modelo de propensión productivo:** SQL/Spark genera features ([03](03-procesamiento-datos.md)) → Pandera valida → `Pipeline` sklearn + LightGBM → MLflow registra métricas y artefacto → CI promueve a "Production" → BentoML/FastAPI sirve → Evidently monitorea drift → Airflow reentrena.
