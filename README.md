# gurobipy_multi_scenario
```python
#!/usr/bin/env python3.11

# Copyright 2025, Gurobi Optimization, LLC

# Facility location: a company currently ships its product from 5 plants to
# 4 warehouses. It is considering closing some plants to reduce costs. What
# plant(s) should the company close, in order to minimize transportation
# and fixed costs?
#
# Since the plant fixed costs and the warehouse demands are uncertain, a
# scenario approach is chosen.
#
# Note that this example is similar to the facility.py example. Here we
# added scenarios in order to illustrate the multi-scenario feature.
#
# Note that this example uses lists instead of dictionaries.  Since
# it does not work with sparse data, lists are a reasonable option.
#
# Based on an example from Frontline Systems:
#   http://www.solver.com/disfacility.htm
# Used with permission.

import gurobipy as gp
from gurobipy import GRB


# Warehouse demand in thousands of units
demand = [15, 18, 14, 20]

# Plant capacity in thousands of units
capacity = [20, 22, 17, 19, 18]

# Fixed costs for each plant
fixedCosts = [12000, 15000, 17000, 13000, 16000]
maxFixed = max(fixedCosts)
minFixed = min(fixedCosts)

# Transportation costs per thousand units
transCosts = [
    [4000, 2000, 3000, 2500, 4500],
    [2500, 2600, 3400, 3000, 4000],
    [1200, 1800, 2600, 4100, 3000],
    [2200, 2600, 3100, 3700, 3200],
]

# Range of plants and warehouses
plants = range(len(capacity))
warehouses = range(len(demand))

# Model
m = gp.Model("multiscenario")

# Plant open decision variables: open[p] == 1 if plant p is open.
open = m.addVars(plants, vtype=GRB.BINARY, obj=fixedCosts, name="open")

# Transportation decision variables: transport[w,p] captures the
# optimal quantity to transport to warehouse w from plant p
transport = m.addVars(warehouses, plants, obj=transCosts, name="trans")

# You could use Python looping constructs and m.addVar() to create
# these decision variables instead.  The following would be equivalent
# to the preceding two statements...
#
# open = []
# for p in plants:
#     open.append(m.addVar(vtype=GRB.BINARY,
#                          obj=fixedCosts[p],
#                          name="open[%d]" % p))
#
# transport = []
# for w in warehouses:
#     transport.append([])
#     for p in plants:
#         transport[w].append(m.addVar(obj=transCosts[w][p],
#                                      name="trans[%d,%d]" % (w, p)))

# The objective is to minimize the total fixed and variable costs
m.ModelSense = GRB.MINIMIZE

# Production constraints
# Note that the right-hand limit sets the production to zero if the plant
# is closed
m.addConstrs(
    (transport.sum("*", p) <= capacity[p] * open[p] for p in plants), "Capacity"
)

# Using Python looping constructs, the preceding would be...
#
# for p in plants:
#     m.addConstr(sum(transport[w][p] for w in warehouses)
#                 <= capacity[p] * open[p], "Capacity[%d]" % p)

# Demand constraints
demandConstr = m.addConstrs(
    (transport.sum(w) == demand[w] for w in warehouses), "Demand"
)

# ... and the preceding would be ...
# for w in warehouses:
#     m.addConstr(sum(transport[w][p] for p in plants) == demand[w],
#                 "Demand[%d]" % w)

# We constructed the base model, now we add 7 scenarios
#
# Scenario 0: Represents the base model, hence, no manipulations.
# Scenario 1: Manipulate the warehouses demands slightly (constraint right
#             hand sides).
# Scenario 2: Double the warehouses demands (constraint right hand sides).
# Scenario 3: Manipulate the plant fixed costs (objective coefficients).
# Scenario 4: Manipulate the warehouses demands and fixed costs.
# Scenario 5: Force the plant with the largest fixed cost to stay open
#             (variable bounds).
# Scenario 6: Force the plant with the smallest fixed cost to be closed
#             (variable bounds).

m.NumScenarios = 7

# Scenario 0: Base model, hence, nothing to do except giving the scenario a
#             name
m.Params.ScenarioNumber = 0
m.ScenNName = "Base model"

# Scenario 1: Increase the warehouse demands by 10%
m.Params.ScenarioNumber = 1
m.ScenNName = "Increased warehouse demands"
for w in warehouses:
    demandConstr[w].ScenNRhs = demand[w] * 1.1

# Scenario 2: Double the warehouse demands
m.Params.ScenarioNumber = 2
m.ScenNName = "Double the warehouse demands"
for w in warehouses:
    demandConstr[w].ScenNRhs = demand[w] * 2.0

# Scenario 3: Decrease the plant fixed costs by 5%
m.Params.ScenarioNumber = 3
m.ScenNName = "Decreased plant fixed costs"
for p in plants:
    open[p].ScenNObj = fixedCosts[p] * 0.95

# Scenario 4: Combine scenario 1 and scenario 3
m.Params.ScenarioNumber = 4
m.ScenNName = "Increased warehouse demands and decreased plant fixed costs"
for w in warehouses:
    demandConstr[w].ScenNRhs = demand[w] * 1.1
for p in plants:
    open[p].ScenNObj = fixedCosts[p] * 0.95

# Scenario 5: Force the plant with the largest fixed cost to stay open
m.Params.ScenarioNumber = 5
m.ScenNName = "Force plant with largest fixed cost to stay open"
open[fixedCosts.index(maxFixed)].ScenNLB = 1.0

# Scenario 6: Force the plant with the smallest fixed cost to be closed
m.Params.ScenarioNumber = 6
m.ScenNName = "Force plant with smallest fixed cost to be closed"
open[fixedCosts.index(minFixed)].ScenNUB = 0.0

# Save model
m.write("multiscenario.lp")

# Guess at the starting point: close the plant with the highest fixed costs;
# open all others

# First open all plants
for p in plants:
    open[p].Start = 1.0

# Now close the plant with the highest fixed cost
p = fixedCosts.index(maxFixed)
open[p].Start = 0.0
print(f"Initial guess: Closing plant {p}\n")

# Use barrier to solve root relaxation
m.Params.Method = 2

# Solve multi-scenario model
m.optimize()

# Print solution for each scenario
for s in range(m.NumScenarios):
    # Set the scenario number to query the information for this scenario
    m.Params.ScenarioNumber = s

    print(f"\n\n------ Scenario {s} ({m.ScenNName})")

    # Check if we found a feasible solution for this scenario
    if m.ModelSense * m.ScenNObjVal >= GRB.INFINITY:
        if m.ModelSense * m.ScenNObjBound >= GRB.INFINITY:
            # Scenario was proven to be infeasible
            print("\nINFEASIBLE")
        else:
            # We did not find any feasible solution - should not happen in
            # this case, because we did not set any limit (like a time
            # limit) on the optimization process
            print("\nNO SOLUTION")
    else:
        print(f"\nTOTAL COSTS: {m.ScenNObjVal:g}")
        print("SOLUTION:")
        for p in plants:
            if open[p].ScenNX > 0.5:
                print(f"Plant {p} open")
                for w in warehouses:
                    if transport[w, p].ScenNX > 0:
                        print(
                            f"  Transport {transport[w, p].ScenNX:g} units to warehouse {w}"
                        )
            else:
                print(f"Plant {p} closed!")

# Print a summary table: for each scenario we add a single summary line
print("\n\nSummary: Closed plants depending on scenario\n")
print(f"{'':8} | {'Plant':>17} {'|':>13}")

tableStr = [f"{'Scenario':8} |"]
tableStr += [f"{p:>5}" for p in plants]
tableStr += [f"| {'Costs':>6}  Name"]
print(" ".join(tableStr))

for s in range(m.NumScenarios):
    # Set the scenario number to query the information for this scenario
    m.Params.ScenarioNumber = s

    tableStr = f"{s:<8} |"

    # Check if we found a feasible solution for this scenario
    if m.ModelSense * m.ScenNObjVal >= GRB.INFINITY:
        if m.ModelSense * m.ScenNObjBound >= GRB.INFINITY:
            # Scenario was proven to be infeasible
            print(tableStr, f"{'infeasible':<30}| {'-':>6}  {m.ScenNName:<}")
        else:
            # We did not find any feasible solution - should not happen in
            # this case, because we did not set any limit (like a time
            # limit) on the optimization process
            print(tableStr, f"{'no solution found':<30}| {'-':>6}  {m.ScenNName:<}")
    else:
        for p in plants:
            if open[p].ScenNX > 0.5:
                tableStr += f" {' ':>5}"
            else:
                tableStr += f" {'x':>5}"

        print(tableStr, f"| {m.ScenNObjVal:6g}  {m.ScenNName:<}")
```


以下是对你提供的 `gurobipy` 多场景优化示例程序的逐行解析，重点讲解 **多场景功能的核心语法** 和实现逻辑：

---

### **1. 模型基础结构**
#### **1.1 初始化模型**
```python
m = gp.Model("multiscenario")
```
- 创建名为 `multiscenario` 的优化模型。

#### **1.2 定义变量**
```python
# 二值变量：open[p]表示是否开启工厂p
open = m.addVars(plants, vtype=GRB.BINARY, obj=fixedCosts, name="open")

# 连续变量：transport[w,p]表示从工厂p到仓库w的运输量
transport = m.addVars(warehouses, plants, obj=transCosts, name="trans")
```
- `open` 变量是二值变量，目标函数系数为固定成本 `fixedCosts`。
- `transport` 变量是连续变量，目标函数系数为运输成本 `transCosts`。

---

### **2. 定义公共约束**
#### **2.1 生产能力约束**
```python
m.addConstrs(
    (transport.sum("*", p) <= capacity[p] * open[p] for p in plants), "Capacity"
)
```
- 每个工厂的总运输量不超过其产能，且当工厂关闭 (`open[p]=0`) 时运输量为0。

#### **2.2 仓库需求约束**
```python
demandConstr = m.addConstrs(
    (transport.sum(w) == demand[w] for w in warehouses), "Demand"
)
```
- 每个仓库的总运输量等于其需求量 `demand[w]`，约束命名为 `Demand`。

---

### **3. 多场景定义**
#### **3.1 总场景数设置**
```python
m.NumScenarios = 7  # 定义7个场景
```
- 关键参数 `NumScenarios` 声明总共有7个不同的场景。

#### **3.2 场景参数修改语法**
每个场景通过以下步骤定义：
1. **设置当前场景编号**：
   ```python
   m.Params.ScenarioNumber = 0  # 切换到场景0
   ```
2. **修改场景特定参数**：
   - **约束右侧值 (RHS)**：
     ```python
     demandConstr[w].ScenNRhs = new_value  # 修改约束右侧
     ```
   - **目标函数系数**：
     ```python
     open[p].ScenNObj = new_cost  # 修改变量的目标系数
     ```
   - **变量边界**：
     ```python
     open[p].ScenNLB = 1.0  # 强制变量下限为1（开启工厂）
     open[p].ScenNUB = 0.0  # 强制变量上限为0（关闭工厂）
     ```
3. **场景命名**：
   ```python
   m.ScenNName = "Base model"  # 为场景命名
   ```

---

### **4. 各场景具体实现**
#### **场景0：基准模型**
```python
m.Params.ScenarioNumber = 0
m.ScenNName = "Base model"
```
- 不做任何修改，使用初始参数。

#### **场景1：需求增加10%**
```python
m.Params.ScenarioNumber = 1
m.ScenNName = "Increased warehouse demands"
for w in warehouses:
    demandConstr[w].ScenNRhs = demand[w] * 1.1  # 修改需求约束右侧
```
- 通过 `ScenNRhs` 修改仓库需求约束的右侧值。

#### **场景5：强制开启固定成本最高的工厂**
```python
m.Params.ScenarioNumber = 5
m.ScenNName = "Force plant with largest fixed cost to stay open"
open[fixedCosts.index(maxFixed)].ScenNLB = 1.0  # 设置变量下限为1
```
- 通过 `ScenNLB` 强制变量取值为1（工厂必须开启）。

---

### **5. 求解与结果提取**
#### **5.1 求解模型**
```python
m.optimize()  # 求解多场景模型
```
- 所有场景会同时求解，找到满足所有场景约束的鲁棒解。

#### **5.2 提取场景结果**
```python
for s in range(m.NumScenarios):
    m.Params.ScenarioNumber = s  # 切换到场景s
    print(f"Scenario {s} ({m.ScenNName})")
    print(f"Total costs: {m.ScenNObjVal}")  # 目标函数值
    print(f"Plant status: {open[p].ScenNX}")  # 变量值
```
- **关键属性**：
  - `ScenNObjVal`: 场景 `s` 的目标函数值。
  - `ScenNX`: 变量在场景 `s` 中的取值。
  - `ScenNName`: 场景名称。

---

### **6. 核心语法总结**
| 功能                | 语法示例                          | 说明                       |
|---------------------|-----------------------------------|----------------------------|
| 设置总场景数        | `m.NumScenarios = 7`             | 必须提前声明               |
| 切换当前场景        | `m.Params.ScenarioNumber = n`    | `n` 从0开始                |
| 修改约束右侧值      | `constr.ScenNRhs = value`        | 仅影响当前场景             |
| 修改变量目标系数    | `var.ScenNObj = value`           | 覆盖全局目标系数           |
| 修改变量边界        | `var.ScenNLB = 1.0` 或 `ScenNUB` | 强制变量取值范围           |
| 获取场景结果        | `var.ScenNX`, `m.ScenNObjVal`    | 需先切换场景编号           |

---

### **7. 关键注意事项**
1. **场景独立性**：
   - 每个场景的参数修改需在对应的 `ScenarioNumber` 下进行。
   - 未修改的参数会沿用全局设置。

2. **性能优化**：
   - 当场景数较多时，使用并行求解：
     ```python
     m.Params.ConcurrentMIP = 8  # 并行线程数
     ```

3. **调试技巧**：
   - 导出模型文件检查场景定义：
     ```python
     m.write("model.lp")  # 导出模型文件
     ```

---

### **8. 示例输出逻辑**
代码最后通过遍历所有场景并打印结果，展示不同场景下的工厂关闭决策和总成本：
```python
for s in range(m.NumScenarios):
    m.Params.ScenarioNumber = s  # 切换场景
    if m.ScenNObjVal < GRB.INFINITY:
        print(f"Scenario {s}: Open plants = {[p for p in plants if open[p].ScenNX > 0.5]}")
```
- 输出每个场景的最优解，直观比较不同场景下的决策差异。

通过这个示例，你可以清晰看到 Gurobi 如何通过多场景功能处理不确定性，并在单一模型中同时优化多个可能的情况。
