# SDH VC3 双向数据流设计

> **适用范围**: STM-1 线路侧与 E3 支路侧之间的 VC3 处理  
> **协议依据**: ITU-T G.707, G.783  
> **设计重点**: 接收方向（线路→支路）与发送方向（支路→线路）完整数据流  
> **版本**: v1.1  
> **日期**: 2026-04-07

---

## 目录

1. [接收方向：线路侧 → 支路侧](#一接收方向线路侧--支路侧)
2. [发送方向：支路侧 → 线路侧](#二发送方向支路侧--线路侧)
3. [TU-3 指针处理详解](#三tu-3-指针处理详解)
4. [双向对照简图](#四双向对照简图)
5. [VC3 处理关键参数对照](#五vc3-处理关键参数对照)
6. [双向差异说明](#六双向差异说明)

---

## 一、接收方向：线路侧 → 支路侧

```mermaid
graph TD
    %% 定义节点样式
    classDef phy fill:#e1f5ff,stroke:#01579b,stroke-width:2px
    classDef soh fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    classDef ho fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    classDef lo fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px
    classDef trib fill:#ffebee,stroke:#c62828,stroke-width:2px
    
    %% 物理层
    OPT_RX["光接收<br/>STM-1 155.52M<br/>G.957"]:::phy --> RSOH
    
    %% 段开销层 - RSOH
    subgraph RSOH_GRP ["RSOH 终端 (G.707 §9.2)"]
        RSOH["RSOH 终端<br/>•A1/A2 帧定位<br/>•B1 BIP-8<br/>•J0 踪迹"]:::soh
    end
    
    RSOH --> MSOH
    
    %% 段开销层 - MSOH
    subgraph MSOH_GRP ["MSOH 终端 (G.707 §9.2)"]
        MSOH["MSOH 终端<br/>•B2 BIP-24<br/>•K1/K2 APS<br/>•S1 同步状态"]:::soh
    end
    
    MSOH --> AU4_PTR
    
    %% 高阶通道层
    subgraph HO_GRP ["高阶通道层 (G.707 §8.1)"]
        AU4_PTR["AU-4 指针解释<br/>•H1/H2 解码 (0-782)<br/>•正负调整处理<br/>•VC-4 定位"]:::ho
    end
    
    AU4_PTR --> TUG3_DEMUX
    
    %% VC-4 解复用
    subgraph DEMUX_GRP ["VC-4 解复用 (G.707 §7.3.7)"]
        TUG3_DEMUX["TUG3 分离器<br/>•判C2结构<br/>•提取3路TUG3<br/>•列4/7/10...259"]:::ho
    end
    
    TUG3_DEMUX -->|"TUG3 #1"| TU3_PTR1
    TUG3_DEMUX -->|"TUG3 #2"| TU3_PTR2
    TUG3_DEMUX -->|"TUG3 #3"| TU3_PTR3
    
    %% 低阶通道层 - TU3 指针（修正版）
    subgraph LO_PTR ["低阶通道层 (G.707 §8.2)"]
        TU3_PTR1["TU-3 指针解释 #1<br/>H1/H2 (0-764)<br/>•I-bits→正调整<br/>•D-bits→负调整(V3)<br/>•NDF→新数据"]:::lo
        TU3_PTR2["TU-3 指针解释 #2<br/>H1/H2 (0-764)<br/>•I-bits→正调整<br/>•D-bits→负调整(V3)<br/>•NDF→新数据"]:::lo
        TU3_PTR3["TU-3 指针解释 #3<br/>H1/H2 (0-764)<br/>•I-bits→正调整<br/>•D-bits→负调整(V3)<br/>•NDF→新数据"]:::lo
    end
    
    TU3_PTR1 --> VC3_1
    TU3_PTR2 --> VC3_2
    TU3_PTR3 --> VC3_3
    
    %% VC-3 提取
    subgraph VC3_GRP ["VC-3 处理"]
        VC3_1["VC-3 #1 提取<br/>9×85=765字节<br/>POH:J2/N2/K3"]:::lo
        VC3_2["VC-3 #2 提取<br/>9×85=765字节<br/>POH:J2/N2/K3"]:::lo
        VC3_3["VC-3 #3 提取<br/>9×85=765字节<br/>POH:J2/N2/K3"]:::lo
    end
    
    VC3_1 --> LPC
    VC3_2 --> LPC
    VC3_3 --> LPC
    
    %% LPC 交叉
    subgraph LPC_GRP ["LPC 低阶交叉 (G.783 §12.3)"]
        LPC["LPC<br/>VC-3 低阶交叉<br/>3×3 矩阵"]:::lo
    end
    
    LPC --> C3_DEMUX1
    LPC --> C3_DEMUX2
    LPC --> C3_DEMUX3
    
    %% C-3 去映射
    subgraph C3_GRP ["C-3 去映射 (G.707 §10)"]
        C3_DEMUX1["C-3 #1 去映射<br/>码速调整<br/>34.368M"]:::trib
        C3_DEMUX2["C-3 #2 去映射<br/>码速调整<br/>34.368M"]:::trib
        C3_DEMUX3["C-3 #3 去映射<br/>码速调整<br/>34.368M"]:::trib
    end
    
    C3_DEMUX1 --> HDB3_DEC1
    C3_DEMUX2 --> HDB3_DEC2
    C3_DEMUX3 --> HDB3_DEC3
    
    %% E3 接口
    subgraph E3_GRP ["E3 支路接口 (G.703 §7)"]
        HDB3_DEC1["HDB3 译码<br/>34.368M"]:::trib --> E3_1["E3 输出 #1"]
        HDB3_DEC2["HDB3 译码<br/>34.368M"]:::trib --> E3_2["E3 输出 #2"]
        HDB3_DEC3["HDB3 译码<br/>34.368M"]:::trib --> E3_3["E3 输出 #3"]
    end
```

### 接收方向处理要点

| 处理阶段 | 功能说明 | 协议依据 |
|---------|---------|---------|
| **光接收** | 光电转换，时钟恢复，串并转换 | G.957 |
| **RSOH终端** | A1/A2帧定位，B1误码监测，J0踪迹 | G.707 §9.2 |
| **MSOH终端** | B2误码监测，K1/K2 APS，S1同步状态 | G.707 §9.2 |
| **AU-4指针** | 解释H1/H2（10位值0-782），正负调整处理 | G.707 §8.1 |
| **TUG3分离** | 根据C2字节判别结构，提取3路TUG3 | G.707 §7.3.7 |
| **TU-3指针** | 解释H1/H2（10位值0-764）：I-bits正调整，D-bits负调整，NDF新数据 | G.707 §8.2 |
| **VC-3提取** | 9行×85列，提取POH（J2/N2/K3） | G.707 §7.3 |
| **LPC交叉** | VC-3级低阶交叉连接（可选） | G.783 §12.3 |
| **C-3去映射** | 码速调整解调，HDB3译码 | G.707 §10, G.703 §7 |

---

## 二、发送方向：支路侧 → 线路侧

```mermaid
graph TD
    %% 定义节点样式
    classDef phy fill:#e1f5ff,stroke:#01579b,stroke-width:2px
    classDef soh fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
    classDef ho fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    classDef lo fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px
    classDef trib fill:#ffebee,stroke:#c62828,stroke-width:2px
    
    %% E3 输入
    subgraph E3_IN ["E3 支路接口 (G.703 §7)"]
        E3_1["E3 输入 #1<br/>34.368M"] --> HDB3_ENC1["HDB3 编码"]
        E3_2["E3 输入 #2<br/>34.368M"] --> HDB3_ENC2["HDB3 编码"]
        E3_3["E3 输入 #3<br/>34.368M"] --> HDB3_ENC3["HDB3 编码"]
    end
    
    HDB3_ENC1 --> C3_MAP1
    HDB3_ENC2 --> C3_MAP2
    HDB3_ENC3 --> C3_MAP3
    
    %% C-3 映射
    subgraph C3_MAP ["C-3 映射 (G.707 §10)"]
        C3_MAP1["C-3 #1 映射<br/>异步映射<br/>码速调整"]:::trib
        C3_MAP2["C-3 #2 映射<br/>异步映射<br/>码速调整"]:::trib
        C3_MAP3["C-3 #3 映射<br/>异步映射<br/>码速调整"]:::trib
    end
    
    C3_MAP1 --> VC3_GEN1
    C3_MAP2 --> VC3_GEN2
    C3_MAP3 --> VC3_GEN3
    
    %% VC-3 生成
    subgraph VC3_GEN ["VC-3 生成"]
        VC3_GEN1["VC-3 #1 形成<br/>9×85=765字节<br/>插入POH:J2/N2/K3"]:::lo
        VC3_GEN2["VC-3 #2 形成<br/>9×85=765字节<br/>插入POH:J2/N2/K3"]:::lo
        VC3_GEN3["VC-3 #3 形成<br/>9×85=765字节<br/>插入POH:J2/N2/K3"]:::lo
    end
    
    VC3_GEN1 --> LPC
    VC3_GEN2 --> LPC
    VC3_GEN3 --> LPC
    
    %% LPC 交叉
    subgraph LPC_TX ["LPC 低阶交叉 (G.783 §12.3)"]
        LPC["LPC<br/>VC-3 低阶交叉<br/>3×3 矩阵"]:::lo
    end
    
    LPC --> TU3_PTR1
    LPC --> TU3_PTR2
    LPC --> TU3_PTR3
    
    %% TU-3 指针生成（修正版）
    subgraph TU3_GEN ["低阶通道层 (G.707 §8.2)"]
        TU3_PTR1["TU-3 指针生成 #1<br/>H1/H2 (0-764)<br/>•I-bits→正调整(H3后填充)<br/>•D-bits→负调整(V3数据)<br/>•NDF→新数据"]:::lo
        TU3_PTR2["TU-3 指针生成 #2<br/>H1/H2 (0-764)<br/>•I-bits→正调整(H3后填充)<br/>•D-bits→负调整(V3数据)<br/>•NDF→新数据"]:::lo
        TU3_PTR3["TU-3 指针生成 #3<br/>H1/H2 (0-764)<br/>•I-bits→正调整(H3后填充)<br/>•D-bits→负调整(V3数据)<br/>•NDF→新数据"]:::lo
    end
    
    TU3_PTR1 --> TUG3_MUX
    TU3_PTR2 --> TUG3_MUX
    TU3_PTR3 --> TUG3_MUX
    
    %% TUG3 复用
    subgraph TUG3 ["VC-4 复用 (G.707 §7.3.7)"]
        TUG3_MUX["TUG3 复用器<br/>3路间插<br/>列4/7/10...259"]:::ho
    end
    
    TUG3_MUX --> VC4_GEN
    
    %% VC-4 生成
    subgraph VC4_FORM ["高阶通道层"]
        VC4_GEN["VC-4 形成<br/>•3×TUG3<br/>•C2信号标签<br/>•POH插入"]:::ho
    end
    
    VC4_GEN --> AU4_PTR
    
    %% AU-4 指针
    subgraph AU4 ["AU-4 指针 (G.707 §8.1)"]
        AU4_PTR["AU-4 指针生成<br/>H1/H2/H3<br/>VC-4定位"]:::ho
    end
    
    AU4_PTR --> MSOH_INS
    
    %% MSOH 插入
    subgraph MSOH ["MSOH 插入 (G.707 §9.2)"]
        MSOH_INS["MSOH 插入<br/>•B2计算<br/>•K1/K2<br/>•S1"]:::soh
    end
    
    MSOH_INS --> RSOH_INS
    
    %% RSOH 插入
    subgraph RSOH ["RSOH 插入 (G.707 §9.2)"]
        RSOH_INS["RSOH 插入<br/>•A1/A2<br/>•B1计算<br/>•扰码"]:::soh
    end
    
    RSOH_INS --> OPT_TX
    
    %% 光发送
    OPT_TX["光发送<br/>STM-1 155.52M<br/>G.957"]:::phy
```

### 发送方向处理要点

| 处理阶段 | 功能说明 | 协议依据 |
|---------|---------|---------|
| **E3输入** | HDB3码型接收，34.368M时钟恢复 | G.703 §7 |
| **C-3映射** | 异步映射，码速调整，插入POH | G.707 §10 |
| **VC-3形成** | 9行×85列结构，生成J2/N2/K3 | G.707 §7.3 |
| **LPC交叉** | VC-3级低阶交叉连接（可选） | G.783 §12.3 |
| **TU-3指针** | 生成H1/H2（10位值0-764）：I-bits正调整（H3后填充），D-bits负调整（V3数据），NDF新数据 | G.707 §8.2 |
| **TUG3复用** | 3路VC-3间插复用到TUG3 | G.707 §7.3.7 |
| **VC-4形成** | 3×TUG3 + POH，C2信号标签 | G.707 §7.3 |
| **AU-4指针** | 生成H1/H2/H3，定位VC-4 | G.707 §8.1 |
| **MSOH插入** | B2计算，K1/K2，S1 | G.707 §9.2 |
| **RSOH插入** | A1/A2，B1计算，扰码 | G.707 §9.2 |

---

## 三、TU-3 指针处理详解

### 3.1 H1/H2 字节结构（G.707 §8.2.2）

```
┌─────────────────────────────────────────┐
│           H1/H2 指针字节结构             │
├─────────┬─────────┬───────────────────┤
│  Bits   │  名称   │      功能         │
├─────────┼─────────┼───────────────────┤
│ 15-12   │ N-bits  │ NDF: 0110=正常    │
│         │         │      1001=新数据  │
├─────────┼─────────┼───────────────────┤
│ 11-10   │ 保留    │ 未使用            │
├─────────┼─────────┼───────────────────┤
│  9-0    │ Pointer │ 指针值: 0-764     │
├─────────┼─────────┼───────────────────┤
│ H1中    │ I-bits  │ 7,5,3,1: 正调整   │
│ H2中    │ D-bits  │ 6,4,2,0: 负调整   │
└─────────┴─────────┴───────────────────┘
```

### 3.2 指针调整处理（G.707 §8.2.3）

```mermaid
graph TD
    subgraph TU3_PTR_DETAIL ["TU-3 指针处理状态机 (G.707 §8.2.6)"]
        H1H2["H1/H2 字节输入<br/>•N-bits[15:12]<br/>•I-bits[7,5,3,1,15]<br/>•D-bits[6,4,2,0,14]<br/>•Pointer[9:0]"]
        
        H1H2 --> NDF_CHECK{NDF检测}
        H1H2 --> I_CHECK{I-bits<br/>5位多数决}
        H1H2 --> D_CHECK{D-bits<br/>5位多数决}
        
        NDF_CHECK -->|1001| NDF_ACT["新数据标志处理<br/>•接受新指针值<br/>•立即更新VC-3位置<br/>•持续3帧"]
        NDF_CHECK -->|0110| NORMAL["正常操作"]
        
        I_CHECK -->|>=3个1| POS_JUST["正调整<br/>•I-bits反转<br/>•H3字节后填充1字节<br/>•指针+1 (模765)"]
        I_CHECK -->|<3个1| NORMAL
        
        D_CHECK -->|>=3个1| NEG_JUST["负调整<br/>•D-bits反转<br/>•V3字节承载数据<br/>•指针-1 (模765)"]
        D_CHECK -->|<3个1| NORMAL
        
        NDF_ACT --> WAIT3["等待3帧<br/>禁止再次调整"]
        POS_JUST --> WAIT3
        NEG_JUST --> WAIT3
        
        WAIT3 --> NORMAL
    end
```

### 3.3 指针调整对照表

| 调整类型 | I-bits状态 | D-bits状态 | 字节处理 | 指针值变化 | 适用场景 |
|---------|-----------|-----------|---------|-----------|---------|
| **无调整** | 正常 | 正常 | V3字节忽略 | 不变 | 时钟同步 |
| **正调整** | 反转(多数=1) | 正常 | H3后填充1字节 | +1 (模765) | VC-3时钟慢于TU-3时钟 |
| **负调整** | 正常 | 反转(多数=1) | V3承载数据 | -1 (模765) | VC-3时钟快于TU-3时钟 |
| **NDF** | 1001 | 忽略 | 立即更新位置 | 加载新值 | 相位跳变/切换 |

---

## 四、双向对照简图

```mermaid
graph LR
    subgraph RX ["接收方向（线路→支路）"]
        RX_STM[STM-1输入] --> RX_RSOH[RSOH终端]
        RX_RSOH --> RX_MSOH[MSOH终端]
        RX_MSOH --> RX_AU4[AU-4指针解释]
        RX_AU4 --> RX_TUG3[TUG3分离]
        RX_TUG3 --> RX_TU3[TU-3指针解释]
        RX_TU3 --> RX_VC3[VC-3提取]
        RX_VC3 --> RX_LPC[LPC交叉]
        RX_LPC --> RX_C3[C-3去映射]
        RX_C3 --> RX_E3[E3输出]
    end
    
    subgraph TX ["发送方向（支路→线路）"]
        TX_E3[E3输入] --> TX_C3[C-3映射]
        TX_C3 --> TX_VC3[VC-3形成]
        TX_VC3 --> TX_LPC[LPC交叉]
        TX_LPC --> TX_TU3[TU-3指针生成]
        TX_TU3 --> TX_TUG3[TUG3复用]
        TX_TUG3 --> TX_VC4[VC-4形成]
        TX_VC4 --> TX_AU4[AU-4指针生成]
        TX_AU4 --> TX_MSOH[MSOH插入]
        TX_MSOH --> TX_RSOH[RSOH插入]
        TX_RSOH --> TX_STM[STM-1输出]
    end
```

---

## 五、VC3 处理关键参数对照

| 处理环节 | 接收方向（线路→支路） | 发送方向（支路→线路） | 协议依据 |
|---------|---------------------|---------------------|---------|
| **E3接口** | HDB3译码，提取34.368M | HDB3编码，输出34.368M | G.703 §7 |
| **C-3映射** | 码速调整，VC-3→E3 | 码速调整，E3→VC-3 | G.707 §10 |
| **VC-3帧** | 9行×85列=765字节 | 9行×85列=765字节 | G.707 §7.3 |
| **VC-3 POH** | 终端处理（J2/N2/K3） | 生成插入（J2/N2/K3） | G.707 §9.3 |
| **LPC交叉** | 输入VC-3交叉到输出 | 输入VC-3交叉到输出 | G.783 §12.3 |
| **TU-3指针** | 解释H1/H2（0-764）：I-bits正调整，D-bits负调整(V3)，NDF新数据 | 生成H1/H2（0-764）：I-bits正调整(H3后填充)，D-bits负调整(V3数据)，NDF新数据 | G.707 §8.2 |
| **TUG3位置** | 解复用：列4,7,10...259 | 复用：列4,7,10...259 | G.707 §7.3.7 |
| **VC-4复用** | 解复用TUG3 | 复用TUG3到VC-4 | G.707 §7 |
| **AU-4指针** | 解释H1/H2（0-782） | 生成H1/H2（0-782） | G.707 §8.1 |

---

## 六、双向差异说明

| 对比项 | 接收方向（线路→支路） | 发送方向（支路→线路） |
|-------|---------------------|---------------------|
| **处理本质** | 解映射 + 解复用 + 终端 | 映射 + 复用 + 生成 |
| **指针处理** | 指针解释（解码）：检测I/D-bits，识别调整类型 | 指针生成（编码）：产生I/D-bits，执行调整 |
| **开销处理** | 开销终端（终结，提取信息） | 开销插入（产生，创建信息） |
| **码型变换** | HDB3译码（线路码→NRZ） | HDB3编码（NRZ→线路码） |
| **时钟方向** | 从线路恢复时钟 | 从系统时钟发送 |
| **正调整处理** | H3字节后检测到填充字节，丢弃 | H3字节后插入填充字节 |
| **负调整处理** | V3字节中提取数据 | V3字节中插入数据 |
| **BIP校验** | B1/B2计算并与接收值比较 | B1/B2计算并插入 |
| **告警方向** | 检测并向下游插入AIS | 根据上游状态或故障插入AIS |

### 关键协议条款

| 协议章节 | 内容说明 | 应用场景 |
|---------|---------|---------|
| **G.707 §8.2** | TU-3指针的10位值（0-764），I-bits正调整，D-bits负调整(V3)，NDF新数据 | 指针处理器设计 |
| **G.783 §12.3.1** | Sn/Sm_A适配功能，处理VC-3到VC-4的适配 | LPC交叉模块 |
| **G.707 §19** | ODU/VC复用结构（SDH域为VC复用） | 复用器设计 |
| **G.707 §7.3.7** | TUG3在VC-4中的位置（列计算公式） | TUG3分离/复用 |

---

## 附录：VC3 处理速率参数

| 参数 | 数值 | 说明 |
|------|------|------|
| E3标称速率 | 34.368 Mbit/s | G.703 §7 |
| E3容差 | ±20 ppm | G.703 |
| C-3容器速率 | 48.960 Mbit/s | G.707 Table 7-2 |
| VC-3速率 | 48.960 Mbit/s | 含POH |
| TU-3速率 | 49.152 Mbit/s | 含指针开销 |
| 每VC-4VC-3数 | 3路 | TUG3 #1/#2/#3 |
| 每STM-1VC-3数 | 3路 | 单AU-4 |
| TU-3指针范围 | 0-764 | 10位值 |
| TU-3调整单位 | 1字节 | 正/负调整均为1字节 |

---

*文档待进一步细化完善*
