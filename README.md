# Qurei-Neural-Network
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "398a1114",
   "metadata": {},
   "source": [
    "# CHAPTER 10 - Quantum Neural Networks - Code"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "d60f20a9",
   "metadata": {},
   "source": [
    "*Note*: You may skip the following six cells if you have alredy installed the right versions of all the libraries mentioned in *Appendix D*. This will likely NOT be the case if you are running this notebook on a cloud service such as Google Colab."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "c31af5f7",
   "metadata": {},
   "outputs": [],
   "source": [
    "pip install scikit-learn==1.2.1"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "f3428229",
   "metadata": {},
   "outputs": [],
   "source": [
    "pip install tensorflow==2.9.1"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "2e9d0203",
   "metadata": {
    "scrolled": false
   },
   "outputs": [],
   "source": [
    "pip install pennylane==0.26"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "8b8485db",
   "metadata": {},
   "outputs": [],
   "source": [
    "pip install qiskit==0.39.2"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "8d18ebd5",
   "metadata": {},
   "outputs": [],
   "source": [
    "pip install qiskit_machine_learning==0.5.0"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "5fce24fb",
   "metadata": {},
   "outputs": [],
   "source": [
    "pip install matplotlib==3.2.2"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "6d229d9c",
   "metadata": {},
   "outputs": [],
   "source": [
    "import pennylane as qml\n",
    "import numpy as np\n",
    "import tensorflow as tf\n",
    "\n",
    "seed = 4321\n",
    "np.random.seed(seed)\n",
    "tf.random.set_seed(seed)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "146e2244",
   "metadata": {},
   "outputs": [],
   "source": [
    "tf.keras.backend.set_floatx('float64')"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "91261677",
   "metadata": {},
   "outputs": [],
   "source": [
    "from sklearn.datasets import load_breast_cancer\n",
    "\n",
    "x,y = load_breast_cancer(return_X_y = True)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "b1912ddd",
   "metadata": {},
   "outputs": [],
   "source": [
    "from sklearn.model_selection import train_test_split\n",
    "\n",
    "x_tr, x_test, y_tr, y_test = train_test_split(\n",
    "    x, y, train_size = 0.8)\n",
    "x_val, x_test, y_val, y_test = train_test_split(\n",
    "    x_test, y_test, train_size = 0.5)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "d3254304",
   "metadata": {},
   "outputs": [],
   "source": [
    "from sklearn.preprocessing import MaxAbsScaler\n",
    "\n",
    "scaler = MaxAbsScaler()\n",
    "x_tr = scaler.fit_transform(x_tr)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "be78524d",
   "metadata": {},
   "outputs": [],
   "source": [
    "x_test = scaler.transform(x_test)\n",
    "x_val = scaler.transform(x_val)\n",
    "\n",
    "# Restrict all the values to be between 0 and 1.\n",
    "x_test = np.clip(x_test, 0, 1)\n",
    "x_val = np.clip(x_val, 0, 1)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "0f0c65b8",
   "metadata": {},
   "outputs": [],
   "source": [
    "from sklearn.decomposition import PCA\n",
    "\n",
    "pca = PCA(n_components = 4)\n",
    "\n",
    "xs_tr = pca.fit_transform(x_tr)\n",
    "xs_test = pca.transform(x_test)\n",
    "xs_val = pca.transform(x_val)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "9c8784d2",
   "metadata": {},
   "outputs": [],
   "source": [
    "from itertools import combinations\n",
    "\n",
    "def ZZFeatureMap(nqubits, data):\n",
    "\n",
    "    # Number of variables that we will load:\n",
    "    # could be smaller than the number of qubits.\n",
    "    nload = min(len(data), nqubits) \n",
    "\n",
    "    for i in range(nload):\n",
    "        qml.Hadamard(i)\n",
    "        qml.RZ(2.0 * data[i], wires = i)\n",
    "\n",
    "    for pair in list(combinations(range(nload), 2)):\n",
    "        q0 = pair[0]\n",
    "        q1 = pair[1]\n",
    "\n",
    "        qml.CZ(wires = [q0, q1])\n",
    "        qml.RZ(2.0 * (np.pi - data[q0]) *\n",
    "            (np.pi - data[q1]), wires = q1)\n",
    "        qml.CZ(wires = [q0, q1])\n",
    "\n",
    "def TwoLocal(nqubits, theta, reps = 1):\n",
    "    \n",
    "    for r in range(reps):\n",
    "        for i in range(nqubits):\n",
    "            qml.RY(theta[r * nqubits + i], wires = i)\n",
    "        for i in range(nqubits - 1):\n",
    "            qml.CNOT(wires = [i, i + 1])\n",
    "    \n",
    "    for i in range(nqubits):\n",
    "        qml.RY(theta[reps * nqubits + i], wires = i)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "d80d8616",
   "metadata": {},
   "outputs": [],
   "source": [
    "state_0 = [[1], [0]]\n",
    "M = state_0 * np.conj(state_0).T"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "32087608",
   "metadata": {},
   "outputs": [],
   "source": [
    "nqubits = 4\n",
    "dev = qml.device(\"default.qubit\", wires=nqubits)\n",
    "\n",
    "def qnn_circuit(inputs, theta):\n",
    "    ZZFeatureMap(nqubits, inputs)\n",
    "    TwoLocal(nqubits = nqubits, theta = theta, reps = 1)\n",
    "    return qml.expval(qml.Hermitian(M, wires = [0]))\n",
    "\n",
    "qnn = qml.QNode(qnn_circuit, dev, interface=\"tf\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "683d8d6a",
   "metadata": {},
   "outputs": [],
   "source": [
    "weights = {\"theta\": 8}\n",
    "qlayer = qml.qnn.KerasLayer(qnn, weights, output_dim=1)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "0e37ed35",
   "metadata": {},
   "outputs": [],
   "source": [
    "model = tf.keras.models.Sequential([qlayer])"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "f7a38281",
   "metadata": {},
   "outputs": [],
   "source": [
    "opt = tf.keras.optimizers.Adam(learning_rate = 0.005)\n",
    "model.compile(opt, loss=tf.keras.losses.BinaryCrossentropy())"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "11d8fde2",
   "metadata": {},
   "outputs": [],
   "source": [
    "earlystop = tf.keras.callbacks.EarlyStopping(\n",
    "    monitor = \"val_loss\", patience = 2, verbose = 1,\n",
    "    restore_best_weights = True)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "23ba91a2",
   "metadata": {},
   "outputs": [],
   "source": [
    "history = model.fit(xs_tr, y_tr, epochs = 50, shuffle = True,\n",
    "    validation_data = (xs_val, y_val),\n",
    "    batch_size = 20, \n",
    "    callbacks = [earlystop])"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "4874ed16",
   "metadata": {},
   "outputs": [],
   "source": [
    "import matplotlib.pyplot as plt\n",
    "\n",
    "def plot_losses(history):\n",
    "    tr_loss = history.history[\"loss\"]\n",
    "    val_loss = history.history[\"val_loss\"]\n",
    "    epochs = np.array(range(len(tr_loss))) + 1\n",
    "    plt.plot(epochs, tr_loss, label = \"Training loss\")\n",
    "    plt.plot(epochs, val_loss, label = \"Validation loss\")\n",
    "    plt.xlabel(\"Epoch\")\n",
    "    plt.legend()\n",
    "    plt.show()\n",
    "\n",
    "plot_losses(history)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "0a4a4339",
   "metadata": {},
   "outputs": [],
   "source": [
    "from sklearn.metrics import accuracy_score\n",
    "\n",
    "tr_acc = accuracy_score(model.predict(xs_tr) >= 0.5, y_tr)\n",
    "val_acc = accuracy_score(model.predict(xs_val) >= 0.5, y_val)\n",
    "test_acc = accuracy_score(model.predict(xs_test) >= 0.5, y_test)\n",
    "\n",
    "print(\"Train accuracy:\", tr_acc)\n",
    "print(\"Validation accuracy:\", val_acc)\n",
    "print(\"Test accuracy:\", test_acc)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "4b01a3cc",
   "metadata": {},
   "outputs": [],
   "source": [
    "nqubits = 4\n",
    "dev = qml.device(\"default.qubit\", wires=nqubits)\n",
    "\n",
    "nreps = 2 \n",
    "weights_dim = qml.StronglyEntanglingLayers.shape(\n",
    "    n_layers = nreps, n_wires = nqubits)\n",
    "nweights = 3 * nreps * nqubits\n",
    "\n",
    "def qnn_circuit_strong(inputs, theta):\n",
    "    \n",
    "    ZZFeatureMap(nqubits, inputs)\n",
    "    theta1 = tf.reshape(theta, weights_dim)\n",
    "    qml.StronglyEntanglingLayers(weights = theta1,\n",
    "                                 wires = range(nqubits))\n",
    "    \n",
    "    return qml.expval(qml.Hermitian(M, wires = [0]))\n",
    "\n",
    "qnn_strong = qml.QNode(qnn_circuit_strong, dev)\n",
    "\n",
    "weights_strong = {\"theta\": nweights}"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "67114113",
   "metadata": {},
   "outputs": [],
   "source": [
    "from qiskit.circuit.library import ZZFeatureMap, TwoLocal\n",
    "\n",
    "nqubits = 3 # We'll do it for three qubits.\n",
    "\n",
    "zzfm = ZZFeatureMap(nqubits, reps = 1)\n",
    "twol = TwoLocal(nqubits, 'ry', 'cx', 'linear', reps = 1)\n",
    "# Change rep(etition)s above to suit your needs."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "bdd94aa2",
   "metadata": {},
   "outputs": [],
   "source": [
    "from qiskit_machine_learning.neural_networks import TwoLayerQNN\n",
    "from qiskit.providers.aer import AerSimulator\n",
    "\n",
    "qnn = TwoLayerQNN(nqubits, feature_map = zzfm, ansatz = twol,\n",
    "                  quantum_instance = AerSimulator(method=\"statevector\"))"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "dcd92c87",
   "metadata": {},
   "outputs": [],
   "source": [
    "qnn.forward(np.random.rand(qnn.num_inputs),\n",
    "            np.random.rand(qnn.num_weights))"
   ]
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": "Python 3 (ipykernel)",
   "language": "python",
   "name": "python3"
  },
  "language_info": {
   "codemirror_mode": {
    "name": "ipython",
    "version": 3
   },
   "file_extension": ".py",
   "mimetype": "text/x-python",
   "name": "python",
   "nbconvert_exporter": "python",
   "pygments_lexer": "ipython3",
   "version": "3.10.8"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 5
}
