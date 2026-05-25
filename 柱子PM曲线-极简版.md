---
name: draw-column-pm-curve
description: |
  绘制选中柱子的PM曲线。
  触发条件：用户提到"输出选中柱子的PM曲线"、"绘制柱子PM曲线"、"柱子PM曲线"等。
---

# 柱子PM曲线绘制

自动提取选中柱子信息并绘制PM曲线。

## 工作流程

### 第一步：获取选中柱子

```python
selected_columns = GetPKPMSelectedMember(MemberKind='柱')
if not selected_columns:
    print("⚠️ 未选中任何柱子，请先在PKPM中选择柱子")
    return
```

### 第二步：提取柱子信息

```python
col = selected_columns[0]

# 几何属性
geom_props = ExtractColumnsGeometryProperty(selected_columns)
section = geom_props['截面'][0]
centerline = geom_props['中心线'][0]

# 材料属性
special_props = ExtractColumnSpecialProperty(selected_columns)
concrete_grade = special_props[col.ID][0]

# 配筋信息
reinf_info = GetColumnReinforcement(IDlist=[col.ID], RealFlr=col.RealFlr)
seg_info = reinf_info[col.ID]
first_seg = list(seg_info.values())[0]

As_B = first_seg[1]
As_H = first_seg[2]
corner_As = first_seg[3]
As_total = As_B + As_H + 4 * corner_As

# 柱子长度
import math
length = math.sqrt(
    (centerline.P1.x - centerline.P0.x)**2 +
    (centerline.P1.y - centerline.P0.y)**2 +
    (centerline.P1.z - centerline.P0.z)**2
)

# 混凝土强度
fc_dict = {'C30': 14.3, 'C35': 16.7, 'C40': 19.1}
fc = fc_dict.get(concrete_grade, 14.3)



```

### 第三步：生成该柱子的基本组合内力，共有四个小步骤

#### 1.调用 GetAllLoadCaseName 获取所有工况名称，然后调用 GetForce 获取内力，读取柱底(截面位置0)和柱顶(截面位置1)单工况内力

```python
all_cases = GetAllLoadCaseName()

print(f"所有工况名称: {all_cases}")


# 读取柱子的单工况内力

forces = GetForce(IDlist=[col.ID], LoadCaseList=all_cases, RealFlr=col.RealFlr, kind='柱')

print(f"\n成功获取柱子内力数据")


# 查看数据结构

print(f"数据结构: {type(forces)}")

print(f"柱子ID: {list(forces.keys())}")


# 获取第一段的数据

seg_data = forces[col.ID]

print(f"段号: {list(seg_data.keys())}")


# 获取第一段的工况数据

first_seg_key = list(seg_data.keys())[0]

case_data = seg_data[first_seg_key]

print(f"工况数: {len(case_data)}")

print(f"工况名称: {list(case_data.keys())[:5]}...")  # 只显示前5个


# 查看一个工况的内力数据结构

first_case = list(case_data.keys())[0]

force_data = case_data[first_case]

print(f"\n工况'{first_case}'的内力数据:")

print(f"  截面位置: {list(force_data.keys())}")

print(f"  截面0的内力: {force_data[0]}")  # [轴力,X剪力,Y剪力,扭矩,X弯矩,Y弯矩]

print(f"  截面1的内力: {force_data[1]}")
```



#### 2.定义基本组合工况,严格按照下面的组合规则进行组合
```python
# 定义基本组合工况
basic_combinations = {}

# 1. 恒载+活载组合
basic_combinations["1.恒+活(恒载有利)"] = [
    ("恒载", 1.0),
    ("活载", 1.5)
]

basic_combinations["1.恒+活(恒载不利)"] = [
    ("恒载", 1.3),
    ("活载", 1.5)
]

# 2. 恒载+活载+风荷载组合（考虑风的正负方向）
wind_cases = ["X向风", "Y向风"]
for wind_case in wind_cases:
    basic_combinations[f"2.{wind_case}(+)"] = [
        ("恒载", 1.3),
        ("活载", 1.5 * 0.7),
        (wind_case, 1.5)
    ]
    basic_combinations[f"2.{wind_case}(-)"] = [
        ("恒载", 1.3),
        ("活载", 1.5 * 0.7),
        (wind_case, -1.5)
    ]

# 3. 恒载+活载+地震组合（考虑地震的正负方向）
earthquake_cases = ["X向地震", "Y向地震", "XY向地震", "YX向地震"]

for eq_case in earthquake_cases:
    basic_combinations[f"3.{eq_case}(+)"] = [
        ("恒载", 1.3),
        ("活载", 1.5 * 0.5),
        (eq_case, 1.4)
    ]
    basic_combinations[f"3.{eq_case}(-)"] = [
        ("恒载", 1.3),
        ("活载", 1.5 * 0.5),
        (eq_case, -1.4)
    ]

print(f"共定义了 {len(basic_combinations)} 个基本组合")


```

#### 3.把基本组合后的内力保存在字典forces，格式：N（kn）Vx(kN) Vy(kN) T(kN·m) Mx(kN·m) My(kN·m)

```python
# 获取柱子的段号
seg_key = list(forces[col.ID].keys())[0]
case_data = forces[col.ID][seg_key]

# 组合内力字典
forces_combined = {}

# 对每个基本组合进行内力组合
for comb_name, comb_items in basic_combinations.items():
    # 初始化柱底和柱顶的内力
    bottom_force = [0.0, 0.0, 0.0, 0.0, 0.0, 0.0]  # [N, Vx, Vy, T, Mx, My]
    top_force = [0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
    
    # 遍历组合中的每个工况
    for case_name, factor in comb_items:
        # 检查工况是否存在
        if case_name in case_data:
            # 获取该工况的内力
            case_force = case_data[case_name]
            
            # 柱底内力（截面位置0）
            bottom_force[0] += case_force[0][0] * factor  # N
            bottom_force[1] += case_force[0][1] * factor  # Vx
            bottom_force[2] += case_force[0][2] * factor  # Vy
            bottom_force[3] += case_force[0][3] * factor  # T
            bottom_force[4] += case_force[0][4] * factor  # Mx
            bottom_force[5] += case_force[0][5] * factor  # My
            
            # 柱顶内力（截面位置1）
            top_force[0] += case_force[1][0] * factor  # N
            top_force[1] += case_force[1][1] * factor  # Vx
            top_force[2] += case_force[1][2] * factor  # Vy
            top_force[3] += case_force[1][3] * factor  # T
            top_force[4] += case_force[1][4] * factor  # Mx
            top_force[5] += case_force[1][5] * factor  # My
    
    # 保存组合后的内力
    forces_combined[comb_name] = {
        'bottom': bottom_force,
        'top':
 top_force
    }

print(f"成功组合了 {len(forces_combined)} 个工况的内力")
print(f"\n示例 - 组合'1.恒+活(恒载不利)'的内力:")
print(f"  柱底: N={forces_combined['1.恒+活(恒载不利)']['bottom'][0]:.2f}kN, "
      f"Mx={forces_combined['1.恒+活(恒载不利)']['bottom'][4]:.2f}kN·m, "
      f"My={forces_combined['1.恒+活(恒载不利)']['bottom'][5]:.2f}kN·m")
print(f"  柱顶: N={forces_combined['1.恒+活(恒载不利)']['top'][0]:.2f}kN, "
      f"Mx={forces_combined['1.恒+活(恒载不利)']['top'][4]:.2f}kN·m, "
      f"My={forces_combined['1.恒+活(恒载不利)']['top'][5]:.2f}kN·m")


```
#### 4.从字典forces中筛选出轴力N（kn） Mx(kN·m) My(kN·m)，并且是 N（kn）和 Mx(kN·m)做为一个字典 ，N（kn）  My(kN·m)做为一个字典 ，并在屏幕上输出该字典

```python
# 筛选轴力N和弯矩Mx、My
# 创建两个字典：N-Mx 和 N-My
forces_N_Mx = {}
forces_N_My = {}

for comb_name, force_data in forces_combined.items():
    # 柱底内力
    N_bottom = -force_data['bottom'][0]
    Mx_bottom = force_data['bottom'][4]
    My_bottom = force_data['bottom'][5]
    
    # 柱顶内力
    N_top = -force_data['top'][0]
    Mx_top = force_data['top'][4]
    My_top = force_data['top'][5]
    
    # 保存到字典
    forces_N_Mx[comb_name] = {
        'bottom': {'N': N_bottom, 'Mx': Mx_bottom},
        'top': {'N': N_top, 'Mx': Mx_top}
    }
    
    forces_N_My[comb_name] = {
        'bottom': {'N': N_bottom, 'My': My_bottom},
        'top': {'N': N_top, 'My': My_top}
    }

# 输出所有组合的轴力N和弯矩Mx、My
print("=" * 20)
print("基本组合内力（轴力N、弯矩Mx、弯矩My）")
print("=" * 20)

for comb_name in forces_combined.keys():
    # 柱底
    N_b = forces_combined[comb_name]['bottom'][0]
    Mx_b = forces_combined[comb_name]['bottom'][4]
    My_b = forces_combined[comb_name]['bottom'][5]
    print(f"{comb_name:<30} {'柱底':<6} 轴力N={N_b:<12.2f} 弯矩Mx={Mx_b:<12.2f} 弯矩My={My_b:<12.2f}")
    
    # 柱顶
    N_t = forces_combined[comb_name]['top'][0]
    Mx_t = forces_combined[comb_name]['top'][4]
    My_t = forces_combined[comb_name]['top'][5]
    print(f"{'':<30} {'柱顶':<6} 轴力N={N_t:<12.2f} 弯矩Mx={Mx_t:<12.2f} 弯矩My={My_t:<12.2f}")
    print("+++++++++++++++++++++++++++++++++")

print("=" * 80)

print({forces_N_Mx})
print({forces_N_My})

```



### 第四步：绘制PM曲线

```python
result = ColPMFigPlot(
    B=str(section.B),
    H=str(section.H),
    As_total=str(As_total),
    As_B=str(As_B),
    As_H=str(As_H),
    fc=str(fc),
    fy='360',
    l=str(length),
    Es='200000',
    ID=str(col.ID)
    forces_N_Mx=forces_N_Mx,
    forces_N_My=forces_N_My
)

print(f"✅ PM曲线绘制完成！")
print(f"图片保存路径: {result}")


# ===== Skill成功标记 =====

print("<skill:pm_curve_plot_success>")
print("author:liuyong")

```






## 执行规则

执行成功后，必须输出：

```text
<skill:pm_curve_plot_success>
```
只输出轴力N和弯矩MX 弯矩MY三个组合内力

## 使用示例

**用户输入：** 输出选中柱子的PM曲线

**输出：** 
```
✅ PM曲线绘制完成！author:liuyong
图片保存路径: D:\...\ColPMM1359.svg
<skill:pm_curve_plot_success>
```
