掌握一门编程语言最好的办法或许是将它的编译器设计出来。毫无疑问那些开发Python编译器的人应该是世界上对Python了解最深刻的人群之一。我用python开发过不少程序，但是每次反思或复盘的时候总是感觉对Python的认知还不到位，由此也看了很多讲Python的书，但看的时候感觉好像懂了，但过了一段时间后又忘了，也就是说单纯看书很难将某一项技术完全内化。当然技能的掌握必然要从实践中来，但是我发现在使用Python开发程序时，我总是使用它的一部分功能就够了，或者说居于我的思维模式限制，我在使用python开发时总是落入一个套路，这使得我只能掌握python技术的冰山一角，就如同井底之蛙一样只了解一小块内容，为了能够打破认知局限，让我自己能更全面的对python的设计原理有更深入的了解，我打算尝试做一个能运行的python编译器。

我在标题中使用了”试用”，也就是这是一个尝试性质，编译器的技术难度足够大，我不清楚能做到哪一步，所以采取了走一步看一步的方式，能做多少就多少，也有可能尝试后发现太难而做不下去，因此是”试用“的由来。

我计划用Go语言来实现python编译器，这样完成这个项目后我们能收获一箭双雕的好处，一是掌握如何使用GO来开发一个复杂程序，一是对python的设计原理能有深入的了解和掌握。首先我们来尝试做一个简单的，基于栈的虚拟机，后面我们会把python编译成字节码，然后在我们设计的虚拟机上运行，这个过程跟java类似。

对虚拟机而言，首先需要字节码，它们是针对虚拟机的一系列操作指令，例如push 1, push 2, add,这三条字节码会把数值1，2压入虚拟机，然后弹出栈顶的两个数值进行相加，把相加结果放到堆栈的顶部，如下图所示：
![请添加图片描述](https://img-blog.csdnimg.cn/7f34d580b87d47f69553fd57acf4245d.png)
首先我们要实现的是字节码，所谓字节码就是一个操作指令，后面跟着0个或1个操作数，例如push 1, add等，每个操作指令用一个数值表示，一旦虚拟机读取到对应的数值时就执行相应操作，如果我们使用0x01表示push，那么当虚拟机读取到数字0x01时，它就会把跟在这个指令后面的4个字节对应的数值压入堆栈。

我们先创建一个文件夹叫GO_Python，然后在里面再创建一个文件夹叫code,接着创建文件code.go，它对应字节码的实现代码，我们先完成一些基本定义：
```
package code 

import (
	"encoding/binary"
	"fmt"
)

type Instructions [] byte //字节码集合

type Opcode byte  //操作码

const (
	opConstant opcode = iota
	OpPop  //弹出堆栈
	OpAdd  //将栈顶两个数弹出并相加，把结果压入堆栈
	OpSub
	OpMul 
	OpDiv
	//后面还有更多操作码需要定义
)
```

假设我们把python代码编译成字节码后，它们就对应Instructions，在字节码中有一些字节代表了特定操作，例如push, add, sub等，这些操作就对应操作码。操作码可以使用不同的数值来区分，因此代码中定义了枚举类型数值来对应操作码，注意到操作码的类型为byte，这意味着我们的虚拟机最多支持128种不同操作。

我们还需要对操作码做进一步描述，例如给定操作码OpPop后，我们希望能找到它对应的字符串，例如"POP"，同时不同操作码其实对应不同的操作数，例如OpAdd就对应两个操作数，所有这些信息我们都需要以代码的方式记录下来，因此我们继续添加如下定义：
```
type Definition struct {
	Name string //操作码对应的字符串
	OperandWidths [] int //操作数对应的字节数
}

var definitions = map[Opcode]*Definition {
	OpConstant: {"OpConstant", []int{2}},
}

func Lookup(op byte) (*Definition, error ) {
	//给定操作码，返回它对应的信息定义
	def, ok := definitions[Opcode(op)]
	if !ok {
		return nil, fmt.Errorf("opcode %d undefined", op)
	}

	return def, nil
}
```
代码中需要说明的一点是operandWidths，它对应操作数的字节长度，例如对于表达式 255 + 1,我们需要使用操作码opAdd将两个数值弹出，然后相加，由于255对应两个字节，1对应1个字节，于是对应的definition就是{“OpAdd", []{2, 1}}。由此我们可以理解上面代码中操作码"OpConstant"对应的操作数有2个字节的长度，OpConstant操作符的作用是在一个常量数组中查找对应数组，它的操作数就是数组下标，我们会把代码中所定义的一切常量都放入到一个特定的常量数组中，相关内容后续我们会进一步解释。

接下来我们看如何将操作码以及操作数转换成一条可以被虚拟机执行的指令，所谓”指令“其实就是byte数组，数组的第一个字节对应操作符的数值，后续字节对应操作数的内容。假设有一条字节码为 OpConstant 65534, 那么将它转换为指令时，第一个字节就对应操作码OpConstant的数值，也就是0，接下来就对应操作数的字节内容，由于65534拆解成两个最字节就是0xFF, 0xFE,于是这条操作码转换为”指令“后就是[]byte{0x0, 0xFF, 0xFE}, 我们看对应的代码实现：
```
func Make(op Opcode, operands ...int) []byte {
	//给定操作码，创建字节码指令
	def , ok := definitions[op]
	if !ok {
		return []byte{}
	}

	//一条指令的字节长度包括操作码对应的长度加上操作数对应的长度
	instructionLen :=1  //操作码长度始终为1
	for _, w := range def.OperandWidths {
		instructionLen += w 
	}
    //一条指令由一系列字节组成,第一个字节就是操作码，接下来的字节对应操作数
	instructions := make([]byte, instructionLen) 
	instructions[0] = op //设置操作码对应的字节
	offset := 1
	for i, o := range operands {
		width := def.OperandWidths[i]
		switch width {
		case 2:
			//把一个16比特数,也就是uint16类型的数值拆解成2个byte放到数组中
			binary.BigEndian.PutUint16(instruction[offset:], uint16(o))
		}
		offset += width 
	}

	return instruction
}
```
于是当我们的虚拟机在执行指令[]byte{0x0, 0xFF, 0xFE}时，它发现第一个字节为0，于是它就知道要执行OpConstant操作，也就是要从常量数组中把对应的内容拿出来，同时根据definitions结构体可以知道，对应的操作数有两个字节，于是它把接下来的两个字节也就是0xFF,0xFE组合起来称为一个操作数，也就是获得了65534这个数值，然后将65534作为数组的下标，从常量数组中把下标为65534的元素取出来。

最好的学习方式就是即时反馈，所以我们有了一些代码后，要尽快把它们运行起来，看看执行结果，这样我们才能通过实验来验证逻辑或者是清除头脑中的疑惑，如果没有即时反馈，那么我们很快就会因为困惑积累过多而放弃努力，因此我们将以测试的方式把上面代码跑起来，在同一目录下创建明文code_test.go的文件，在里面添加测试代码如下：
```
package code 
import "testing"

func TestMake(t *testing.T) {
	tests := []struct {
		op Opcode 
		operands []int 
		expected []byte 
	} {
		{OpConstant, []int{65534}, []byte{0x0, 0xFF, 0xFE}},
	}

	for _, tt := range tests {
		instruction := Make(tt.op, tt.operands...)
		if len(instruction) != len(tt.expected) {
			t.Errorf("instruction has wrong length. want=%d, got=%d",
		len(tt.expected), len(instruction))
		}

		for i , b := range tt.expected {
			if instruction[i] != tt.expected[i] {
				t.Errorf("wrong byte at pos %d, want=%d, got = %d", i, b, instruction[i])
			}
		}
	}
}
```
完成上面代码后，进入到code目录然后执行：
```
go  test
```
这样就能将测试用例跑起来，通过结果可以看到用例能通过，也就是Make函数准确的将操作码及其对应的操作数转换成了一条指令字节数组，为了好消化，我们一次不要搞太多，先在这里停止。


