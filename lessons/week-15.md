# Week 15: Neural Network Fundamentals with PyTorch

Phase 4 begins. General knowledge; these fundamentals are stable.

## Goal

Write a training loop from scratch and explain every line.

## Ideas

| Term | Meaning |
| --- | --- |
| Tensor | An n-dimensional array that can live on a GPU and track gradients |
| Parameter | A number the model learns (weights and biases) |
| Forward pass | Compute predictions from inputs |
| Loss | A number measuring how wrong the predictions are |
| Gradient | How the loss changes if a parameter changes slightly |
| Backpropagation | Computing all gradients with the chain rule |
| Optimiser | Rule for updating parameters from gradients (SGD, Adam, AdamW) |
| Learning rate | Step size of each update |
| Batch | Examples processed together |
| Epoch | One pass over the training set |
| Overfitting | Good on training data, poor on new data |

## Minimal training loop

```python
import torch
from torch import nn
from torch.utils.data import DataLoader, TensorDataset

torch.manual_seed(0)
device = "cuda" if torch.cuda.is_available() else "cpu"

# toy data: 2 features -> 2 classes
X = torch.randn(1000, 2)
y = (X[:, 0] * X[:, 1] > 0).long()
train = DataLoader(TensorDataset(X[:800], y[:800]), batch_size=32, shuffle=True)
val = DataLoader(TensorDataset(X[800:], y[800:]), batch_size=64)

model = nn.Sequential(nn.Linear(2, 16), nn.ReLU(), nn.Linear(16, 2)).to(device)
loss_fn = nn.CrossEntropyLoss()
opt = torch.optim.AdamW(model.parameters(), lr=1e-2)

for epoch in range(20):
    model.train()
    for xb, yb in train:
        xb, yb = xb.to(device), yb.to(device)
        logits = model(xb)              # forward
        loss = loss_fn(logits, yb)      # loss
        opt.zero_grad()                 # clear old gradients
        loss.backward()                 # backpropagation
        opt.step()                      # update parameters

    model.eval()
    correct = total = 0
    with torch.no_grad():               # no gradients for evaluation
        for xb, yb in val:
            pred = model(xb.to(device)).argmax(dim=1).cpu()
            correct += (pred == yb).sum().item()
            total += len(yb)
    print(epoch, f"train_loss={loss.item():.3f} val_acc={correct/total:.3f}")
```

The five lines in the inner loop (forward, loss, zero_grad, backward, step)
are the whole of training. Everything else is bookkeeping.

## Things to understand, not memorise

- **Why non-linearity** (ReLU): without it, stacked linear layers collapse
  into one linear layer.
- **Why three splits**: train to learn, validation to tune, test to report
  once at the end.
- **Reading loss curves**: training loss falling while validation loss rises
  means overfitting; both flat and high means underfitting or a learning
  rate problem.
- **Learning rate**: too high and loss explodes or bounces; too low and it
  barely moves.
- **Regularisation**: dropout, weight decay, early stopping, more data.
- **`model.train()` and `model.eval()`**: switch layers such as dropout
  between modes.

## Do

1. Run the loop above. Change the hidden size, the learning rate and remove
   the ReLU; note what happens each time.
2. Plot training and validation loss per epoch.
3. Train a small classifier on a real dataset (for example a sentiment set
   with bag-of-words features).
4. Overfit on purpose with 20 training examples, then fix it.
5. Save and load the model with `torch.save(model.state_dict(), ...)`.

## Resource

Andrej Karpathy, "Neural Networks: Zero to Hero" (free videos). Watch the
first two and code along.

## Checkpoint

- [ ] You can write the training loop without looking.
- [ ] You can explain a loss curve that shows overfitting.
- [ ] You can say what backpropagation computes.

## Interview questions

1. What does `loss.backward()` do?
2. Why do we call `optimizer.zero_grad()`?
3. How do you detect and reduce overfitting?
4. Why do neural networks need activation functions?
