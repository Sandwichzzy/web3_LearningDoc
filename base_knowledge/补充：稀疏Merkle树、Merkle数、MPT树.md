### 1. 首先，理解普通的Merkle树

想象一下，你有8个数据块（比如交易记录）。为了确保这些数据没有被篡改，你构建了一棵树：

- **叶子节点**：在最底层，每个数据块都经过哈希计算，得到一个哈希值（就像它的数字指纹）。
- **父节点**：每两个相邻的叶子节点的哈希值拼接在一起，再计算一次哈希，得到它们父节点的哈希值。
- **重复这个过程**：继续向上，每两个相邻的父节点哈希再拼接、再哈希，直到树顶，得到一个最终的根哈希。

这个根哈希就像是整个数据集的“唯一摘要”。只要任何一个数据块被修改，根哈希就会彻底改变。

**普通Merkle树的问题**：如果你想证明“某个数据块**不存在**”于这个集合中，是非常困难的。你几乎需要展示所有数据来证明它真的不在里面。

------

### 2. 稀疏Merkle树是什么？

稀疏Merkle树解决了“证明不存在”的问题。

**核心思想：** 它是一棵巨大的、预先建好的“空树”。

- **巨大的地址空间**：想象这棵树有2²⁵⁶个叶子节点！这个数量远远超过宇宙中的原子数。每个叶子节点都有一个唯一的“地址”（从0到2²⁵⁶-1）。
- **初始状态为空**：在最开始，整棵树都是空的。每一个叶子节点都被赋予了一个默认的“空值”的哈希（比如全为0的哈希）。
- **“稀疏”的含义**：因为地址空间巨大，而实际存储的数据很少（比如只有几个数据块），所以这棵树的绝大多数叶子节点都是空的，只有极少数叶子有数据。这就是“稀疏”的由来。

------

### 3. 稀疏Merkle树是如何实现的？

我们一步步来看它的操作逻辑。

#### 步骤一：确定数据的位置

在稀疏Merkle树中，一个数据存放在哪个叶子节点，不是由顺序决定的，而是由它的 **“键”** 决定的。

- 比如，你的数据键是 `Alice`。我们通过一个哈希函数（如SHA-256）把 `Alice` 映射到一个256位的二进制数，比如 `0101...110`。
- 这个256位的二进制数，就是 `Alice` 这个数据在巨大树中的**地址**。它指明了从树根到叶子需要走的路径（0向左，1向右）。

#### 步骤二：插入和更新树

现在我们要把 `Alice` 的数据存入树中。

1. **定位叶子**：根据 `Alice` 的哈希地址，我们沿着路径（比如 0->1->0->1...）一路找到对应的那个叶子节点。
2. **更新叶子**：将这个空叶子节点更新为 `Alice` 数据的哈希值。
3. **向上递归更新**：由于这个叶子节点的哈希值变了，它的父节点、祖父节点……一直到树根的哈希值，都需要沿着刚才的路径**重新计算**。
4. **最终**：你得到了一个新的根哈希，它代表了包含 `Alice` 数据后的整个树的状态。

**关键点**：即使树中只有 `Alice` 一个数据，整棵树的根哈希也是基于所有2²⁵⁶个叶子节点（其中绝大多数是空节点）计算出来的。

#### 步骤三：如何实现高效存储和计算？

你可能会想：“存储一棵有2²⁵⁶个节点的树？这根本不可能！”

确实如此。所以实现上使用了**优化技巧**，核心是 **“懒计算”** 和 **“节点共享”**。

- **默认哈希**：我们预先知道，所有空叶子节点的哈希都是 `H_zero`（比如全0）。
- **空子树的根哈希**：我们也可以预先计算出一棵完全为空的大树的根哈希是多少。实际上，每一棵空子树（无论多大）的根哈希都可以被预先计算并缓存起来。
- **实际存储**：我们只存储和计算那些因为插入了数据而**发生改变的节点路径**。其余的绝大部分节点，我们都不需要实际存储，因为我们知道它们就是默认的空哈希。

**比喻**：
想象一片巨大的空白方格纸（代表空树）。当你只在其中一个格子里点了一个黑点（插入数据），你只需要记录下这个黑点的位置和它影响的极少数线条。整张纸的其他部分，你知道它们都是白色的，不需要一一描述。

#### 步骤四：成员证明 vs. 非成员证明

这是稀疏Merkle树的杀手级应用。

- **成员证明**：证明 `Alice` 的数据存在。
  - 和普通Merkle树一样，提供一条从 `Alice` 的叶子节点到树根的“认证路径”（即路径上所有兄弟节点的哈希值）。验证者可以通过计算验证这条路径最终能否得到正确的根哈希。
- **非成员证明**：证明 `Bob` 的数据**不存在**。
  1. 根据 `Bob` 的键找到它本应存在的叶子节点。
  2. 提供通往这个叶子节点的认证路径。
  3. 如果这个路径的末端（即 `Bob` 的叶子节点）的哈希值是**默认的空值哈希**（`H_zero`），那么就铁证如山地证明了 `Bob` 的数据不存在于这棵树中。

------

### 总结与优势

**稀疏Merkle树是什么？**
它是一个拥有巨大、固定数量叶子节点的Merkle树，其中绝大部分叶子为空，并填充默认值。数据通过其键的哈希值被定位到特定的叶子节点上。

**它是如何实现的？**

1. 定义一个巨大的地址空间（如2²⁵⁶）和一套默认的空哈希值。
2. 通过数据的键确定其在树中的唯一路径。
3. 只创建和更新被数据影响的路径上的节点，其余部分用预计算的空哈希代替，从而实现高效存储和计算。
4. 利用从叶子到根路径上的兄弟节点哈希值，可以同时提供**存在性**和**非存在性**证明。

**核心优势：**

- **高效的非成员证明**：这是它最主要的价值。
- **计算的连续性**：树的形态是固定的，只有叶子节点的值在变。这使得计算增量更新（比如从状态A到状态B的根哈希变化）非常高效。

**典型应用：**
在区块链中，用于存储整个系统的状态（比如每个账户的余额），当一个新节点加入时，你可以快速向它证明某个账户不存在，而不需要下载整个历史记录。



### 4. MPT 与 “经典”稀疏Merkle树的联系

为什么说 MPT 和稀疏Merkle树有联系呢？因为它们要解决的核心问题是相同的：**如何高效地、可验证地存储一个巨大的、稀疏的键值对集合。**

- **键**：以太坊账户的地址（20字节）。
- **值**：账户的余额、Nonce、代码哈希、存储根等。
- **稀疏性**：地址空间是 2¹⁶⁰，这是一个天文数字，但实际存在的活跃账户数量相对极少。所以这是一个典型的稀疏数据集。

“经典”的稀疏Merkle树（我们之前讨论的那种）会使用一个完整的二进制树，并利用默认哈希来优化空节点。而 MPT 采用了一种不同的、更显式的优化策略。

### 5. MPT 是如何实现“稀疏”和优化的？

MPT 没有使用固定的二进制深度和默认哈希，而是通过其节点类型来动态地、紧凑地表示这棵稀疏的树：

1. **空节点**：直接表示一个空。
2. **分支节点**：一个拥有17个元素的数组（[0...f] + 一个value）。用于处理键在某个节点处开始分叉的情况。
3. **扩展节点**：用于“压缩”路径。当一个节点只有一个子节点时，使用扩展节点将多个连续的单路径节点合并成一个，存储 `[共享前缀, 下一个节点的指针]`。这极大地缩短了树的深度和寻径时间。
4. **叶子节点**：存储最终的数据。形式为 `[键的剩余部分, 值]`。

**通过这种设计，MPT 实现了与稀疏Merkle树相似的目标，但方式不同：**

| 特性           | 经典稀疏Merkle树                               | 以太坊的 MPT                                                 |
| :------------- | :--------------------------------------------- | :----------------------------------------------------------- |
| **非成员证明** | 通过提供一条以**默认空哈希**结尾的路径来证明。 | 通过提供一条**路径不存在**的证明来证明。例如，路径指向一个空节点，或者路径中的一个分支节点在关键位置没有子节点。 |
| **效率优化**   | 依赖于计算和缓存大量**默认哈希**。             | 依赖于**路径压缩** 和显式的节点类型，避免遍历不必要的深度。  |
| **结构**       | **固定的、完整的**二叉树结构。                 | **动态的、不规则的**树结构，形状取决于存储的数据。           |
| **键处理**     | 键的整个哈希值直接决定了从根到叶的完整路径。   | 键的哈希（或原始地址）被逐个半字节地                         |



```
package main

import (

  "crypto/sha256"

  "encoding/hex"

  "fmt"

  "math/big"

)

*// ==================== 稀疏Merkle树 (Sparse Merkle Tree) ====================*

*// SparseMerkleTree 稀疏Merkle树结构 高度固定*

type SparseMerkleTree struct {

  Root    []byte       *// 根哈希*

  Height   int        *// 树的高度*

  DefaultHash []byte       *// 默认空节点哈希*

  Nodes    map[string][]byte *// 存储非空节点 key -> hash*

  Leaves   map[string][]byte *// 存储叶子节点 key -> value*

}

*// NewSparseMerkleTree 创建新的稀疏Merkle树*

func NewSparseMerkleTree(height int) *SparseMerkleTree {

  smt := &SparseMerkleTree{

​    Height: height,

​    Nodes:  make(map[string][]byte),

​    Leaves: make(map[string][]byte),

  }

  

  *// 计算默认空节点哈希（从叶子到根）*

  emptyHash := sha256.Sum256([]byte{})

  smt.DefaultHash = emptyHash[:]

  

  *// 初始化根哈希为默认哈希*

  smt.Root = smt.DefaultHash

  

  return smt

}

*// hash 计算哈希值*

func (smt *SparseMerkleTree) hash(data []byte) []byte {

  h := sha256.Sum256(data)

  return h[:]

}

*// hashChildren 计算两个子节点的父节点哈希*

func (smt *SparseMerkleTree) hashChildren(left, right []byte) []byte {

  combined := append(left, right...)

  return smt.hash(combined)

}

*// keyToPath 将键转换为二进制路径*

func (smt *SparseMerkleTree) keyToPath(key []byte) []bool {

  *// 将键转换为大整数*

  keyInt := new(big.Int).SetBytes(key)

  

  *// 转换为二进制路径（从根到叶子）*

  path := make([]bool, smt.Height)

  for i := 0; i < smt.Height; i++ {

​    *// 从最高位开始读取*

​    path[i] = keyInt.Bit(smt.Height-1-i) == 1

  }

  

  return path

}

*// Insert 插入或更新键值对*

func (smt *SparseMerkleTree) Insert(key, value []byte) {

  *// 1. 将键转换为路径*

  path := smt.keyToPath(key)

  

  *// 2. 计算值的哈希作为叶子节点*

  valueHash := smt.hash(value)

  

  *// 3. 存储叶子节点*

  keyStr := hex.EncodeToString(key)

  smt.Leaves[keyStr] = value

  

  *// 4. 从叶子向上更新到根*

  currentHash := valueHash

  

  for depth := smt.Height - 1; depth >= 0; depth-- {

​    *// 获取兄弟节点的哈希*

​    siblingHash := smt.getSiblingHash(key, depth, path)

​    

​    *// 根据路径方向计算父节点哈希*

​    if path[depth] {

​      *// 当前节点在右边*

​      currentHash = smt.hashChildren(siblingHash, currentHash)

​    } else {

​      *// 当前节点在左边*

​      currentHash = smt.hashChildren(currentHash, siblingHash)

​    }

​    

​    *// 存储中间节点哈希*

​    nodeKey := smt.getNodeKey(key, depth)

​    smt.Nodes[nodeKey] = currentHash

  }

  

  *// 5. 更新根哈希*

  smt.Root = currentHash

  

  fmt.Printf("✓ 插入成功: key=%x, value=%s\n", key, string(value))

}

*// Get 查询键对应的值*

func (smt *SparseMerkleTree) Get(key []byte) ([]byte, bool) {

  keyStr := hex.EncodeToString(key)

  value, exists := smt.Leaves[keyStr]

  return value, exists

}

*// Delete 删除键值对*

func (smt *SparseMerkleTree) Delete(key []byte) bool {

  keyStr := hex.EncodeToString(key)

  

  *// 检查键是否存在*

  if _, exists := smt.Leaves[keyStr]; !exists {

​    return false

  }

  

  *// 删除叶子节点*

  delete(smt.Leaves, keyStr)

  

  *// 重新计算根哈希（使用默认哈希替代被删除的节点）*

  path := smt.keyToPath(key)

  currentHash := smt.DefaultHash

  

  for depth := smt.Height - 1; depth >= 0; depth-- {

​    siblingHash := smt.getSiblingHash(key, depth, path)

​    

​    if path[depth] {

​      currentHash = smt.hashChildren(siblingHash, currentHash)

​    } else {

​      currentHash = smt.hashChildren(currentHash, siblingHash)

​    }

​    

​    nodeKey := smt.getNodeKey(key, depth)

​    smt.Nodes[nodeKey] = currentHash

  }

  

  smt.Root = currentHash

  fmt.Printf("✓ 删除成功: key=%x\n", key)

  return true

}

*// getSiblingHash 获取兄弟节点的哈希*

func (smt *SparseMerkleTree) getSiblingHash(key []byte, depth int, path []bool) []byte {

  *// 构造兄弟节点的键*

  siblingKey := make([]byte, len(key))

  copy(siblingKey, key)

  

  *// 翻转指定深度的位*

  byteIdx := depth / 8

  bitIdx := uint(depth % 8)

  if byteIdx < len(siblingKey) {

​    siblingKey[byteIdx] ^= (1 << (7 - bitIdx))

  }

  

  *// 查找兄弟节点的哈希*

  nodeKey := smt.getNodeKey(siblingKey, depth)

  if hash, exists := smt.Nodes[nodeKey]; exists {

​    return hash

  }

  

  *// 如果兄弟节点不存在，返回默认哈希*

  return smt.DefaultHash

}

*// getNodeKey 生成节点的唯一标识*

func (smt *SparseMerkleTree) getNodeKey(key []byte, depth int) string {

  return fmt.Sprintf("%x:%d", key[:min(len(key), 4)], depth)

}

*// GenerateProof 生成Merkle证明*

func (smt *SparseMerkleTree) GenerateProof(key []byte) [][]byte {

  path := smt.keyToPath(key)

  proof := make([][]byte, smt.Height)

  

  *// 收集从叶子到根的所有兄弟节点哈希*

  for depth := smt.Height - 1; depth >= 0; depth-- {

​    siblingHash := smt.getSiblingHash(key, depth, path)

​    proof[smt.Height-1-depth] = siblingHash

  }

  

  return proof

}

*// VerifyProof 验证Merkle证明*

func (smt *SparseMerkleTree) VerifyProof(key, value []byte, proof [][]byte) bool {

  if len(proof) != smt.Height {

​    return false

  }

  

  path := smt.keyToPath(key)

  currentHash := smt.hash(value)

  

  *// 从叶子向根计算哈希*

  for i := 0; i < smt.Height; i++ {

​    siblingHash := proof[i]

​    

​    if path[smt.Height-1-i] {

​      *// 当前节点在右边*

​      currentHash = smt.hashChildren(siblingHash, currentHash)

​    } else {

​      *// 当前节点在左边*

​      currentHash = smt.hashChildren(currentHash, siblingHash)

​    }

  }

  

  *// 比较计算出的根哈希与树的根哈希*

  return compareBytes(currentHash, smt.Root)

}

*// GetRootHash 获取根哈希*

func (smt *SparseMerkleTree) GetRootHash() string {

  return hex.EncodeToString(smt.Root)

}

*// PrintTree 打印树的信息*

func (smt *SparseMerkleTree) PrintTree() {

  fmt.Println("\n" + "=".repeat(70))

  fmt.Println("稀疏Merkle树信息")

  fmt.Println("=".repeat(70))

  fmt.Printf("树高度:    %d\n", smt.Height)

  fmt.Printf("根哈希:    %s\n", smt.GetRootHash())

  fmt.Printf("叶子节点数:  %d\n", len(smt.Leaves))

  fmt.Printf("总节点数:   %d\n", len(smt.Nodes))

  fmt.Printf("默认哈希:   %x\n", smt.DefaultHash)

  

  if len(smt.Leaves) > 0 {

​    fmt.Println("\n叶子节点列表:")

​    for keyStr, value := range smt.Leaves {

​      fmt.Printf("  %s -> %s\n", keyStr[:16]+"...", string(value))

​    }

  }

  

  fmt.Println("=".repeat(70))

}

*// ==================== 工具函数 ====================*

func compareBytes(a, b []byte) bool {

  if len(a) != len(b) {

​    return false

  }

  for i := range a {

​    if a[i] != b[i] {

​      return false

​    }

  }

  return true

}

func min(a, b int) int {

  if a < b {

​    return a

  }

  return b

}

*// 字符串重复函数（Go 1.20+可以直接使用 strings.Repeat）*

type repeatHelper string

func (s repeatHelper) repeat(count int) string {

  result := ""

  for i := 0; i < count; i++ {

​    result += string(s)

  }

  return result

}

func (s string) repeat(count int) string {

  return repeatHelper(s).repeat(count)

}

*// ==================== 测试演示 ====================*

func main() {

  fmt.Println("╔════════════════════════════════════════════════════════════════════╗")

  fmt.Println("║        稀疏Merkle树 (Sparse Merkle Tree) 演示        ║")

  fmt.Println("╚════════════════════════════════════════════════════════════════════╝")

  

  *// 创建稀疏Merkle树（高度为8，支持256个键）*

  fmt.Println("\n【步骤1: 创建稀疏Merkle树】")

  smt := NewSparseMerkleTree(8)

  fmt.Printf("✓ 创建成功，树高度: %d\n", smt.Height)

  

  *// 插入数据*

  fmt.Println("\n【步骤2: 插入键值对】")

  smt.Insert([]byte{1}, []byte("Alice"))

  smt.Insert([]byte{2}, []byte("Bob"))

  smt.Insert([]byte{5}, []byte("Charlie"))

  smt.Insert([]byte{10}, []byte("David"))

  

  *// 打印树信息*

  smt.PrintTree()

  

  *// 查询数据*

  fmt.Println("\n【步骤3: 查询数据】")

  testKeys := [][]byte{{1}, {2}, {3}, {5}}

  for _, key := range testKeys {

​    if value, found := smt.Get(key); found {

​      fmt.Printf("✓ 键 %d 的值: %s\n", key[0], string(value))

​    } else {

​      fmt.Printf("✗ 键 %d 不存在\n", key[0])

​    }

  }

  

  *// 更新数据*

  fmt.Println("\n【步骤4: 更新数据】")

  oldRoot := smt.GetRootHash()

  smt.Insert([]byte{2}, []byte("Bob_Updated"))

  newRoot := smt.GetRootHash()

  fmt.Printf("更新前根哈希: %s\n", oldRoot[:32]+"...")

  fmt.Printf("更新后根哈希: %s\n", newRoot[:32]+"...")

  

  *// 生成和验证证明*

  fmt.Println("\n【步骤5: 生成Merkle证明】")

  key := []byte{5}

  value, _ := smt.Get(key)

  proof := smt.GenerateProof(key)

  

  fmt.Printf("为键 %d 生成证明:\n", key[0])

  fmt.Printf("  值: %s\n", string(value))

  fmt.Printf("  证明长度: %d\n", len(proof))

  fmt.Printf("  证明路径: ")

  for i, p := range proof {

​    if i < 3 {

​      fmt.Printf("%x... ", p[:4])

​    }

  }

  fmt.Println()

  

  *// 验证证明*

  fmt.Println("\n【步骤6: 验证Merkle证明】")

  isValid := smt.VerifyProof(key, value, proof)

  fmt.Printf("证明验证结果: %t\n", isValid)

  

  *// 测试错误的证明*

  wrongValue := []byte("Wrong_Value")

  isValidWrong := smt.VerifyProof(key, wrongValue, proof)

  fmt.Printf("错误值的证明验证结果: %t\n", isValidWrong)

  

  *// 删除数据*

  fmt.Println("\n【步骤7: 删除数据】")

  deleteKey := []byte{10}

  success := smt.Delete(deleteKey)

  if success {

​    fmt.Printf("✓ 删除键 %d 成功\n", deleteKey[0])

​    fmt.Printf("删除后根哈希: %s\n", smt.GetRootHash()[:32]+"...")

  }

  

  *// 验证删除*

  if _, found := smt.Get(deleteKey); !found {

​    fmt.Printf("✓ 确认键 %d 已被删除\n", deleteKey[0])

  }

  

  *// 最终状态*

  fmt.Println("\n【步骤8: 最终状态】")

  smt.PrintTree()

  

  *// 演示稀疏性*

  fmt.Println("\n【步骤9: 演示稀疏性】")

  fmt.Printf("理论最大容量: 2^%d = %d 个键\n", smt.Height, 1<<smt.Height)

  fmt.Printf("实际存储数量: %d 个键\n", len(smt.Leaves))

  fmt.Printf("空间利用率:  %.2f%%\n", float64(len(smt.Leaves))/float64(1<<smt.Height)*100)

  

  fmt.Println("\n✓ 稀疏Merkle树演示完成")

}
```

