---
title: "拥有基本 IO 功能的控制台计算器"
date: 2017-12-29 19:37:04
tags: [cpp, misc]
---

第一次写这么长的代码（虽然里面注释什么的水分比较多), 还是蛮有成就感的. 同时这也是我第一次写这么详细的注释233. 毕竟这还是作业对吧. 

<!-- more -->

```cpp
#include <iostream>
#include <fstream>
#include <string>
#include <sstream>
#include <vector>
#include <cstdlib>
#include <iomanip>
#include <direct.h>

#define WRITE 1
#define DELETE 2
#define EXIT 3
#define CALC 4
#define HELP 5
#define SAVE 6
#define LOAD 7
#define RENAME 8
#define LIST 9
#define READ 10

using namespace std;

/*************************************************
*Function: str2int(longlong作为函数名太长)
*Description: 将string类型转换为long long类型
*************************************************/
long long str2int(const string &string_temp)
{
	long long int_temp;
	stringstream stream(string_temp);
	stream >> int_temp;
	return int_temp;
}

/*************************************************
*Function: int2str(longlong作为函数名太长)
*Description: 将long long类型转换为string类型
*************************************************/
string int2str(long long int_temp)
{
	string string_temp;
	stringstream stream;
	stream << int_temp;
	stream >> string_temp;
	return string_temp;
}

class Rational_Numbers
{
private:
	friend class MyioEngine;//声明为友元,让其可以访问分子分母

	string name;//名字
	long long numerator;//分子
	long long denominator;//分母

	/*************************************************
	*Function: myabs
	*Description: 求绝对值
	*************************************************/
	long long myabs(long long input) 
	{
		if (input > 0)
		{
			return input;
		}
		else
		{
			return -input;
		}
	}

public:

	/*************************************************
	*Description: 四种构造函数 并且输入后自动化简
	*************************************************/
	Rational_Numbers(string _name, long long _numerator, long long _denominator) :numerator(_numerator), denominator(_denominator), name(_name) { simplify(); }; 
	Rational_Numbers(long long _numerator, long long _denominator) :numerator(_numerator), denominator(_denominator), name("default") { simplify(); }; 
	Rational_Numbers(long long _numerator) :numerator(_numerator), denominator(1), name("default") { simplify(); };
	Rational_Numbers():numerator(1), denominator(1), name("default") {};//默认构造函数

	/*************************************************
	\*Description: 重载+ - \* / << >> 符号
	*************************************************/
	Rational_Numbers operator + (Rational_Numbers &input) //重载+号
	{
		 Rational_Numbers temp(numerator\*input.denominator+input.numerator\*denominator,denominator*input.denominator); 
		 return temp.simplify();
	}

	Rational_Numbers operator - (Rational_Numbers &input) //重载-号
	{
		Rational_Numbers temp(numerator\*input.denominator - input.numerator\*denominator, denominator*input.denominator);
		return temp.simplify();
	}

	Rational_Numbers operator * (Rational_Numbers &input) //重载*号
	{
		Rational_Numbers temp(numerator\*input.numerator, denominator\*input.denominator);
		return temp.simplify();
	}

	Rational_Numbers operator / (Rational_Numbers &input) //重载/号
	{
		Rational_Numbers temp(numerator\*input.denominator, denominator\*input.numerator);
		return temp.simplify();
	}

	friend ostream& operator << (ostream &os,Rational_Numbers &input) //重载<<号
	{
		cout << int2str(input.numerator) + " /" + int2str(input.denominator);
		return os;
	}

	friend istream& operator >> (istream &is, Rational_Numbers &input) //重载>>号
	{
		cin >> input.numerator >> input.denominator;
		input.simplify();
		return is;
	}

	bool operator == (Rational_Numbers &input) //重载==号
	{
		simplify();
		input.simplify();
		if (denominator == input.denominator && numerator == input.numerator)
		{
			return true;
		}
		else
		{
			return false;
		}
	}
	
	Rational_Numbers operator = (Rational_Numbers input) //重载=号
	{
		input.simplify();
		name = input.name;
		denominator = input.denominator;
		numerator = input.numerator;
		return *this;
	}
	
	/*************************************************
	*Function: realNumber
	*Description: 返回这个类的实数
	*************************************************/

	double realNumber()
	{
		return static_cast<double> (numerator) / static_cast<double> (denominator);
	}

	/*************************************************
	*Function: simplify
	*Description: 将这个有理数化简
	*************************************************/
	Rational_Numbers simplify()
	{
		if (myabs(numerator) == myabs(denominator) && numerator != 0) //如果分子分母相同, 化简为1
		{
			numerator /= myabs(numerator); //这样能保证符号和原来的相同
			denominator /= myabs(denominator);
		}
		else
		{
			if (numerator == 0) //如果分子为0, 直接返回并将分母设为1
			{
				denominator = 1;
			}
			else
			{
				if ((myabs(numerator) > myabs(denominator)) && (myabs(numerator) % myabs(denominator) == 0)) //如果分子分母是倍数关系, 可以直接化简
				{
					numerator /= myabs(denominator);
					denominator /= myabs(denominator);
				}
				else
				{
					if ((myabs(denominator) > myabs(numerator)) && (myabs(denominator) % myabs(numerator) == 0))
					{
						denominator /= myabs(numerator);
						numerator /= myabs(numerator);
					}
					else
					{
						long long gcd = 0;//Greatest Common Divisor(GCD) 最大公约数
						long long temp = 0;
						long long left = 0;
						long long right = 0;

						if (myabs(numerator) > myabs(denominator)) //大的数作为left
						{
							left = myabs(numerator);
							right = myabs(denominator);
						}
						else
						{
							left = myabs(denominator);
							right = myabs(numerator);
						}

						while (true)  //辗转相减法:left - right = temp 直到 right == temp
						{
							temp = left - right;
							if (temp == right)
							{
								gcd = temp;
								break;
							}
							else
							{
								if (right > temp) //如果temp>right ,根据辗转相减, 大的放左边
								{
									left = right;
									right = temp;
								}
								else
								{
									left = temp;
								}
							}
						}

						numerator /= gcd;
						denominator /= gcd;
					}			
				}			
			}		
		}
		
		if (numerator < 0 && denominator < 0) //如果都小于零, 返回正值
		{
			denominator *= -1;
			numerator *= -1;
		}

		if (numerator > 0 && denominator < 0) //将负号保持在分子上
		{
			numerator *= -1;
			denominator *= -1;
		}

		return *this;
	}
};

class MyioEngine
{
private:
	int mode = 0;//保存当前的模式
	bool wrong = true;//用来检测calcEngine的状态
	vector<string> instructions;//保存指令
	vector<Rational_Numbers> nums;//保存所有的变量

	/*************************************************
	*Function: format
	*Description: 将输入的字符串格式化.大写转小写以及清除空格
	*************************************************/
	void format(string &input) //将字符全部转换为小写,去除空格
	{
		for (int i = input.size() - 1; i >= 0; --i) //逆序以防止删除错误的字符
		{
			if (input[i] >= 'A' && input[i] <= 'Z')
			{
				input[i] += 32;
			}
			if (input[i] == ' ')
			{
				input.erase(input.begin() + i);
			}
		}
	}

	/*************************************************
	*Function: loadInstructions
	*Description: 从控制台录入指令
	*************************************************/
	void loadInstructions() //读取指令
	{
		instructions.clear();
		string instructionLine;//定义命令行
		string instruction;//定义指令
		
		getline(cin, instructionLine);//获取指令行
		for (unsigned int i = 0; i < instructionLine.size(); ++i)
		{
			if (instructionLine[i] != ' ')//第一个循环, 找到第一个不是空格的字符
			{
				for (; i < instructionLine.size(); ++i)//进入第二个循环,检测空格以找到字符串尾部
				{
					if (instructionLine[i] == ' ' || i == instructionLine.size() - 1)//找到第二个空格或者说此字符是指令行的最后一个, 说明字符串到头了
					{
						if (instructionLine[i] != ' ')//兼容此字符是指令行的最后一个的情况
						{
							instruction.push_back(instructionLine[i]);
						}
						format(instruction);
						instructions.push_back(instruction);//放入指令容器中
						instruction.clear();//清除当前指令, 录入下一个
						break;
					}
					instruction.push_back(instructionLine[i]);//不是空格, 将此字符录入指令中
				}
			}
		}
	}

	/*************************************************
	*Function: checkNums
	*Description: 检查输入的字符串是否都是数字
	*************************************************/
	bool checkNums(string input) //检查是否全部是数字
	{
		if (input[0] == '-') //检测第一个数是不是-号
		{
			input.erase(input.begin());//如果是, 删掉-号
		}

		if (input.size() == 0) //检测字符串是不是只有一个负号
		{
			return false;
		}

		for (unsigned int i = 0; i < input.size(); ++i)
		{
			if (input[i] > '9' || input[i] < '0') //只要检测到不在0~9之间的字符就返回false
			{
				return false;
			}
		}
		return true;
	}
	
	/*************************************************
	*Function: checkVaild
	*Description: 检查输入的字符串是否合法
	*************************************************/
	bool checkVaild(string input) 
	{
		for (unsigned int i = 0; i < input.size(); ++i)
		{
			char temp = input[i];
			if (temp == '@' || temp == '$' || temp == '+' || temp == '-' || temp == '*' || temp == '/' || checkNums(input)) //只要检测到非法字符就返回false PS:$运算中要用到, 作为保留字符
			{
				return false;
			}
		}
		return true;
	}

	/*************************************************
	*Function: getAddress
	*Description: 输入变量的名字, 存在返回nums中的地址, 不存在返回-1
	*************************************************/
	int getAddress(string input) //检测是否有相同名字的元素出现
	{
		for (unsigned int i = 0; i < nums.size(); ++i)
		{
			if (input == nums[i].name)
			{
				return i; //相同的返回下标
			}
		}
		return -1; //没有相同的返回-1
	}
	
	/*************************************************
	*Function: getDataDir
	*Description: 得到当前程序的目录位置
	*************************************************/
	string getDataDir()
	{
		char _path[1000];
		_getcwd(_path, 1000);
		string path = _path;
		path = path + "\\vars.RNS"; //加上储存的文件名
		return path;
	}

	/*************************************************
	*Function: deleteTemps
	*Description: 根据计算的次数删掉对应次数的临时变量
	*************************************************/
	void deleteTemps(int tempCount)
	{
		for (int i = 0; i < tempCount; ++i)//删掉计算过程中的中间变量
		{
			nums.pop_back();
		}
	}

	/*************************************************
	*Function: getLeftname
	*Description: 根据input字符串和运算符地址, 找到左边的变量标识符
	*************************************************/
	string getLeftname(string input, int operatorAdd,int &leftCharAdd, string currentOperator)
	{
		string leftname;

		for (leftCharAdd = operatorAdd - 1; leftCharAdd >= 0; --leftCharAdd) //得到运算符左边的标识符的起始字符的地址
		{
			if (input[leftCharAdd] == '+' || input[leftCharAdd] == '-' || input[leftCharAdd] == '*' || input[leftCharAdd] == '/' || input[leftCharAdd] == '@') //遇到计算符号退出, 得到变量的第一个字符的地址
			{
				break;
			}
		}

		++leftCharAdd;//如果通过遇到计算符号退出, j指向的是计算符；若是运行到0退出, j为-1.综上, j需要+1才行


		if (operatorAdd == leftCharAdd)//如果位置相同, 说明运算符左边没有变量的标识符
		{
			wrong = true;
			cout << "表达式中" + currentOperator + "需要一个左值" << endl;
			return leftname;//这个值不会被用到
		}
		else
		{
			leftname.assign(input.begin() + leftCharAdd, input.begin() + operatorAdd);//将名字赋值给leftname
		}
		return leftname;
	}

	/*************************************************
	*Function: getRightname
	*Description: 根据input字符串和运算符地址, 找到右边的变量标识符
	*************************************************/
	string getRightname(string input, int operatorAdd, int &rightCharAdd, string currentOperator)
	{
		string rightname;

												//不想看到warning, 干脆强行转换一波吧233
		for (rightCharAdd = operatorAdd + 1; rightCharAdd < static_cast<int>(input.size()); rightCharAdd++) //得到运算符右边的名字
		{
			if (input[rightCharAdd] == '+' || input[rightCharAdd] == '-' || input[rightCharAdd] == '*' || input[rightCharAdd] == '/' || input[rightCharAdd] == '@') //遇到计算符号退出, 得到变量的第一个字符的地址
			{
				break;
			}
		}

		--rightCharAdd;//如果通过遇到计算符号退出, rightCharAdd指向的是计算符；若是运行到0退出, rightCharAdd为size大小.综上, rightCharAdd需要-1才行

		if (operatorAdd == rightCharAdd)
		{
			wrong = true;
			cout << "表达式中" + currentOperator + "需要一个右值" << endl; //如果位置相同, 说明运算符右边没有变量的标识符
			return rightname;//这个值不会被用到
		}
		else
		{
			rightname.assign(input.begin() + operatorAdd + 1, input.begin() + rightCharAdd + 1);//将名字赋值给rightname,注意此时operatorAdd指向的是计算符号, 所以还得+1
		}
		return rightname;
	}

	/*************************************************
	*Function: negativeConversion
	*Description: 将负号转换为计算器能理解的变量
	*************************************************/
	void negativeConversion(string &input,int &tempCount)
	{
		vector<int> operators;
		nums.emplace_back("$neg", -1, 1);//添加值为-1的变量
		++tempCount;

		for (int i = input.size() - 1; i > 0; --i)//如果减号前面跟着一个运算符号, 那么它的意义是负号.倒序替换适配连续减号的情况
		{
			if (input[i] == '-' && (input[i - 1] == '*' || input[i - 1] == '/' || input[i - 1] == '+' || input[i - 1] == '-') && i != input.size() - 1)
			{
				input.replace(input.begin() + i, input.begin() + i + 1, "$neg@");
			}
		}

		if (input[0] == '-')//如果第一个字符就是减号, 那么减号代表的意义一定是负号
		{
			input.replace(input.begin(), input.begin() + 1, "$neg@");
		}
	}

	/*************************************************
	*Function: calcEngine
	*Description: 计算引擎 输入表达式返回结果
	*************************************************/
	Rational_Numbers calcEngine(string input) 
	{	
		wrong = true;//wrong复位
		Rational_Numbers result("1", 1, 1);
		Rational_Numbers temp("$temp", 1, 1);//创建一个temp变量, 注意在计算中它的名字是temp.name
		vector<int> operators;//记录所有运算符的位置
		int tempCount = 0;//记录运算次数
		int operatorAdd = 0;//记录当前计算运算符的位置

		negativeConversion(input, tempCount);//将意义为负号的减号转换成neg@来计算

		for (unsigned int i = 0; i < input.size(); ++i) //如果input中没有计算符号, 说明输入有误
		{
			if (input[i] == '*' || input[i] == '/' || input[i] == '+' || input[i] == '-' || input[i] == '@')
			{
				wrong = false;
			}
		}

		if (wrong)
		{
			cout << "输入的表达式没有计算符号" << endl;
			deleteTemps(tempCount);
			return result;//这里这个值不会被用到, 只是为了满足返回的要求
		}

		while (true)
		{
			if (input == "$temp" + int2str(tempCount - 1)) //如果只剩下这个, 说明计算完成
			{
				result = nums.back();
				deleteTemps(tempCount);
				return result;
			}

			operators.clear();//复位

			for (unsigned int i = 0; i < input.size(); ++i)
			{
				if (input[i] == '@')//找到除号
				{
					operatorAdd = i;
					goto negative;
				}
			}

			for (unsigned int i = 0; i < input.size(); ++i)
			{				
				if (input[i] == '*' || input[i] == '/' || input[i] == '+' || input[i] == '-')
				{
					operators.push_back(i);//录入所有运算符的位置
				}
			}

			for (unsigned int i = 0; i < operators.size(); ++i) //先找*运算符, 然后依次找/ + - .满足* / + -的顺序进行计算
			{
				if ((input[operators[i]] == '*' && i == 0) || (input[operators[i]] == '*' && input[operators[i - 1]] != '/'))//找到乘号, 如果（是第一个运算符）或者（不是第一个运算符且前面不是除号）, 进行计算.（这么做是为了保证如果只剩下乘和除的情况下, 按照顺序计算）
				{							
					operatorAdd = operators[i];
					goto muti;//跳到计算步骤
				}
			}

			for (unsigned int i = 0; i < input.size(); ++i)
			{
				if (input[i] == '/')//找到除号
				{																											 
					operatorAdd = i;
					goto divide;
				}
			}

			for (unsigned int i = 0; i < operators.size(); ++i)
			{
				if ((input[operators[i]] == '+' && i == 0) || (input[operators[i]] == '+' && input[operators[i - 1]] != '-'))//找到加号, 如果（是第一个运算符）或者（不是第一个运算符且前面不是减号）, 进行计算.（这么做是为了保证如果只剩下加和减的情况下, 按照顺序计算）
				{
					operatorAdd = operators[i];
					goto pls;//plus遇到重名的了, 只好用pls
				}
			}

			for (unsigned int i = 0; i < input.size(); ++i)
			{
				if (input[i] == '-')//找到减号
				{
					operatorAdd = i;
					goto mins;
				}
			}

			muti:
			if (input[operatorAdd] == '*')
			{
				int rightCharAdd;
				int leftCharAdd;
				string leftname = getLeftname(input, operatorAdd, leftCharAdd, "*");//得到左边变量的名字
				string rightname = getRightname(input, operatorAdd, rightCharAdd, "*");//得到左边变量的名字
				
				if (wrong)//寻找名字时出错的跳出
				{
					deleteTemps(tempCount);////删除临时变量
					return result;//这个返回值不会被用到
				}

				if (checkNums(leftname))//如果左边变量是个纯数字, 说明进行实数计算, 建立一个同名的变量
				{
					nums.emplace_back(leftname, str2int(leftname), 1);
					++tempCount;
				}

				if (checkNums(rightname))//如果右边变量是个纯数字, 说明进行实数计算, 建立一个同名的变量
				{
					nums.emplace_back(rightname, str2int(rightname), 1);
					++tempCount;
				}

				int leftadd = getAddress(leftname);//通过标识符得到变量在nums中的地址
				int rightadd = getAddress(rightname);

				if (leftadd == -1)//如果没有找到
				{
					wrong = true;
					cout << "没有找到标识符为" << leftname << "的变量" << endl;
					deleteTemps(tempCount);//删除临时变量
					return result;
				}

				if (rightadd == -1)
				{
					wrong = true;
					cout << "没有找到标识符为" << rightname << "的变量" << endl;
					deleteTemps(tempCount);//删除临时变量
					return result;
				}

				temp = nums[leftadd] * nums[rightadd]; //计算

				temp.name = "$temp" + int2str(tempCount);
				nums.push_back(temp); //$tempX在nums的最后一个, 将其赋值为这两个元素的积
				input.replace(input.begin() + leftCharAdd, input.begin() + rightCharAdd + 1, "$temp" + int2str(tempCount)); //计算完成后将表达式中已经计算的部分替换为$temp方便下一步计算	
				++tempCount;
				continue;
			}

			//========================================================================================================================================================================================//
			//==============================================================================接下来的都跟乘法一样, 只是换了运算符号方法===================================================================//
			//========================================================================================================================================================================================//

			divide:
			if (input[operatorAdd] == '/') 
			{
				int rightCharAdd;
				int leftCharAdd;
				string leftname = getLeftname(input, operatorAdd, leftCharAdd, "/");//得到左边变量的名字
				string rightname = getRightname(input, operatorAdd, rightCharAdd, "/");//得到左边变量的名字

				if (wrong)//寻找名字时出错的跳出
				{
					deleteTemps(tempCount);//删除临时变量
					return result;//这个返回值不会被用到
				}

				if (checkNums(leftname))//如果左边变量是个纯数字, 说明进行实数计算, 建立一个同名的变量, 值为变量名
				{
					nums.emplace_back(leftname, str2int(leftname), 1);
					++tempCount;
				}

				if (checkNums(rightname))//如果右边变量是个纯数字, 说明进行实数计算, 建立一个同名的变量
				{
					nums.emplace_back(rightname, str2int(rightname), 1);
					++tempCount;
				}

				int leftadd = getAddress(leftname);//通过标识符得到变量在nums中的地址
				int rightadd = getAddress(rightname);

				if (leftadd == -1)//如果没有找到
				{
					wrong = true;
					cout << "没有找到标识符为" << leftname << "的变量" << endl;
					deleteTemps(tempCount);//删除临时变量
					return result;
				}

				if (rightadd == -1)
				{
					wrong = true;
					cout << "没有找到标识符为" << rightname << "的变量" << endl;
					deleteTemps(tempCount);//删除临时变量
					return result;
				}

				temp = nums[leftadd] / nums[rightadd]; //计算

				temp.name = "$temp" + int2str(tempCount);
				nums.push_back(temp); //$tempX在nums的最后一个, 将其赋值为这两个元素的商
				input.replace(input.begin() + leftCharAdd, input.begin() + rightCharAdd + 1, "$temp" + int2str(tempCount)); //计算完成后将表达式中已经计算的部分替换为$temp方便下一步计算	
				++tempCount;
				continue;
			}

			pls:
			if (input[operatorAdd] == '+')
			{
				int rightCharAdd;
				int leftCharAdd;
				string leftname = getLeftname(input, operatorAdd, leftCharAdd, "+");//得到左边变量的名字
				string rightname = getRightname(input, operatorAdd, rightCharAdd, "+");//得到左边变量的名字

				if (wrong)//寻找名字时出错的跳出
				{
					deleteTemps(tempCount);////删除临时变量
					return result;//这个返回值不会被用到
				}

				if (checkNums(leftname))//如果左边变量是个纯数字, 说明进行实数计算, 建立一个同名的变量
				{
					nums.emplace_back(leftname, str2int(leftname), 1);
					++tempCount;
				}

				if (checkNums(rightname))//如果右边变量是个纯数字, 说明进行实数计算, 建立一个同名的变量
				{
					nums.emplace_back(rightname, str2int(rightname), 1);
					++tempCount;
				}

				int leftadd = getAddress(leftname);//通过标识符得到变量在nums中的地址
				int rightadd = getAddress(rightname);

				if (leftadd == -1)//如果没有找到
				{
					wrong = true;
					cout << "没有找到标识符为" << leftname << "的变量" << endl;
					deleteTemps(tempCount);//删除临时变量
					return result;
				}

				if (rightadd == -1)
				{
					wrong = true;
					cout << "没有找到标识符为" << rightname << "的变量" << endl;
					deleteTemps(tempCount);//删除临时变量
					return result;
				}

				temp = nums[leftadd] + nums[rightadd]; //计算

				temp.name = "$temp" + int2str(tempCount);
				nums.push_back(temp); //$tempX在nums的最后一个, 将其赋值为这两个元素的和
				input.replace(input.begin() + leftCharAdd, input.begin() + rightCharAdd + 1, "$temp" + int2str(tempCount)); //计算完成后将表达式中已经计算的部分替换为$temp方便下一步计算	
				++tempCount;
				continue;
			}

			mins:
			if (input[operatorAdd] == '-')
			{
				int rightCharAdd;
				int leftCharAdd;
				string leftname = getLeftname(input, operatorAdd, leftCharAdd, "-");//得到左边变量的名字
				string rightname = getRightname(input, operatorAdd, rightCharAdd, "-");//得到左边变量的名字

				if (wrong)//寻找名字时出错的跳出
				{
					deleteTemps(tempCount);////删除临时变量
					return result;//这个返回值不会被用到
				}

				if (checkNums(leftname))//如果左边变量是个纯数字, 说明进行实数计算, 建立一个同名的变量
				{
					nums.emplace_back(leftname, str2int(leftname), 1);
					++tempCount;
				}

				if (checkNums(rightname))//如果右边变量是个纯数字, 说明进行实数计算, 建立一个同名的变量
				{
					nums.emplace_back(rightname, str2int(rightname), 1);
					++tempCount;
				}

				int leftadd = getAddress(leftname);//通过标识符得到变量在nums中的地址
				int rightadd = getAddress(rightname);

				if (leftadd == -1)//如果没有找到
				{
					wrong = true;
					cout << "没有找到标识符为" << leftname << "的变量" << endl;
					deleteTemps(tempCount);//删除临时变量
					return result;
				}

				if (rightadd == -1)
				{
					wrong = true;
					cout << "没有找到标识符为" << rightname << "的变量" << endl;
					deleteTemps(tempCount);//删除临时变量
					return result;
				}

				temp = nums[leftadd] - nums[rightadd]; //计算

				temp.name = "$temp" + int2str(tempCount);
				nums.push_back(temp); //$tempX在nums的最后一个, 将其赋值为这两个元素的差
				input.replace(input.begin() + leftCharAdd, input.begin() + rightCharAdd + 1, "$temp" + int2str(tempCount)); //计算完成后将表达式中已经计算的部分替换为$temp方便下一步计算	
				++tempCount;
				continue;
			}

		negative:
			if (input[operatorAdd] == '@')
			{
				int rightCharAdd;
				int leftCharAdd;
				string leftname = getLeftname(input, operatorAdd, leftCharAdd, "-");//得到左边变量的名字
				string rightname = getRightname(input, operatorAdd, rightCharAdd, "-");//得到左边变量的名字

				if (wrong)//寻找名字时出错的跳出
				{
					deleteTemps(tempCount);//删除临时变量
					return result;//这个返回值不会被用到
				}

				if (checkNums(leftname))//如果左边变量是个纯数字, 说明进行实数计算, 建立一个同名的变量, 值为变量名
				{
					nums.emplace_back(leftname, str2int(leftname), 1);
					++tempCount;
				}

				if (checkNums(rightname))//如果右边变量是个纯数字, 说明进行实数计算, 建立一个同名的变量
				{
					nums.emplace_back(rightname, str2int(rightname), 1);
					++tempCount;
				}

				int leftadd = getAddress(leftname);//通过标识符得到变量在nums中的地址
				int rightadd = getAddress(rightname);

				if (leftadd == -1)//如果没有找到
				{
					wrong = true;
					cout << "没有找到标识符为" << leftname << "的变量" << endl;
					deleteTemps(tempCount);//删除临时变量
					return result;
				}

				if (rightadd == -1)
				{
					wrong = true;
					cout << "没有找到标识符为" << rightname << "的变量" << endl;
					deleteTemps(tempCount);//删除临时变量
					return result;
				}

				temp = nums[leftadd] * nums[rightadd]; //计算

				temp.name = "$temp" + int2str(tempCount);
				nums.push_back(temp); //$tempX在nums的最后一个, 将其赋值为这两个元素的积
				input.replace(input.begin() + leftCharAdd, input.begin() + rightCharAdd + 1, "$temp" + int2str(tempCount)); //计算完成后将表达式中已经计算的部分替换为$temp方便下一步计算	
				++tempCount;
				continue;
			}
		}
	}

public:
	/*************************************************
	*Function: start
	*Description: 启动ioEngine, 与用户交互
	*************************************************/
	void start()
	{
		cout << "Rational Number Calculator On Win32" << endl;
		cout << "Type \"help\" for more information." << endl;

		while (true)
		{
			cout << ">>>";//提示等待用户输入

			mode = 0; //将模式位归零

			loadInstructions(); //读取指令, 不同的指令将mode换成对应的数字

			if (instructions[0] == "write") //done
			{
				mode = WRITE;
			}
			if (instructions[0] == "delete") //done
			{
				mode = DELETE;
			}
			if (instructions[0] == "exit") //done
			{
				mode = EXIT;
			}
			if (instructions[0] == "calc") //done
			{
				mode = CALC;
			}
			if (instructions[0] == "help") //done
			{
				mode = HELP;
			}
			if (instructions[0] == "save") //done
			{
				mode = SAVE;
			}
			if (instructions[0] == "load") //done
			{
				mode = LOAD;
			}
			if (instructions[0] == "rename") //done
			{
				mode = RENAME;
			}			
			if (instructions[0] == "list") //done
			{
				mode = LIST;
			}
			if (instructions[0] == "read") //done
			{
				mode = READ;
			}


			if (mode == 0) //提示输入错误
			{
				cout << "没有" << instructions[0] << "这个指令,输入help获得帮助"<<endl;
				continue;
			}

			switch (mode)
			{
			//================================================//
			//====================WRITE=======================//
			//================================================//
			case WRITE: 							
				if (instructions.size() == 4) //检查参数数量是否正确
				{							  //format: write name numerator denominator	
					if (checkNums(instructions[2]) && checkNums(instructions[3]) && instructions[3]!="0") //是否都是数字,且分母不能为0
					{
						if (checkVaild(instructions[1]))//检查名字是否合法
						{
							int add = getAddress(instructions[1]); //获取这个名字的变量的下标
							if (add != -1) //检测是否有同名的元素
							{
								nums[add].numerator = str2int(instructions[2]);
								nums[add].denominator = str2int(instructions[3]);
								nums[add].simplify(); //化简
								cout << "覆盖成功" << endl;
							}
							else
							{
								nums.emplace_back(instructions[1], str2int(instructions[2]), str2int(instructions[3]));
								cout << "创建成功" << endl;
							}
						}
						else
						{
							cout << "标识符非法" << endl;
						}
					}
					else
					{
						cout << "请输入正确的数字, 且分母不能为0" << endl;
					}
				}
				else
				{
					if (instructions.size() == 3) //检查参数数量是否正确
					{							  //format: write newname anothername
						int add = getAddress(instructions[2]);//检查原变量是否存在
						if (add != -1)
						{
							if (checkVaild(instructions[1]))//检查新变量的标识符是否合法
							{
								int newadd = getAddress(instructions[1]);//检测是否有同名的元素
								if (newadd != -1)
								{
									string newname = nums[newadd].name;
									nums[newadd] = nums[add];//过程中名字被nums[add]的取代
									nums[newadd].name = newname;
									cout << "覆盖成功" << endl;
								}
								else
								{
									Rational_Numbers result;
									result = nums[add];
									result.name = instructions[1];								
									nums.push_back(result);
									cout << "创建成功" << endl;
								}
							}
							else
							{
								cout << "输入的标识符非法" << endl;
							}

						}
						else
						{
							cout << "没有找到名为" << instructions[2] << "的变量" << endl;
						}
					}
					else
					{
						cout << "请输入正确数量的参数" << endl;
					}
				}

				break;
			//================================================//
			//====================DELETE======================//
			//================================================//
			case DELETE: //format: delete name
				if (instructions.size() == 2) //检查参数数量是否正确
				{
					int add = getAddress(instructions[1]); //获取地址
					if (add != -1)//因为重载了等号, erase会删除错误
					{
						nums.erase(nums.begin() + add);
						cout << "删除成功" << endl;
					}
					else
					{
						cout << "没有此标识符的变量" << endl;
					}
				}
				else
				{
					cout << "输入正确数量的参数" << endl;
				}

				break;
			//================================================//
			//=====================EXIT=======================//
			//================================================//
			case EXIT:
				if (instructions.size() == 1) //fotmat: exit
				{
					return; //这里break跳出的是switch,所以用return
				}
				else
				{
					cout << "输入正确数量的参数" << endl;
				}

				break;
			//================================================//
			//=====================CALC=======================//
			//================================================//
			case CALC:
				if (instructions.size() == 3) //fotmat: calc name expression
				{			
					bool isHave$ = false;
					for (unsigned int i = 0; i < instructions[2].size(); ++i)
					{
						if (instructions[2][i] == '

 
)
						{
							isHave$ = true;
						}
					}
					if (!isHave$)//不含有$才能进行下一步, 避免用户输入$tempX导致bug
					{
						int add = getAddress(instructions[1]);
						if (add != -1) //如果存在这个名字的变量, 将其赋值为结果
						{
							Rational_Numbers result = calcEngine(instructions[2]);
							if (!wrong)//确认表达式计算正确
							{
								string name = nums[add].name;
								nums[add] = result;
								nums[add].name = name;
								cout << "覆盖成功" << endl;
							}
							else//计算过程出错就直接退出
							{
								break;
							}
						}
						else//如果不存在这个名字的变量, 创建一个新的变量
						{					
							if (checkVaild(instructions[1]))//检查新变量名字的合法性
							{
								Rational_Numbers result = calcEngine(instructions[2]);
								result.name = instructions[1];
								if (!wrong)//确认表达式计算正确
								{
									nums.push_back(result);
									cout << "变量创建成功" << endl;
								}
								else//计算过程出错就直接退出
								{
									break;
								}
							}
							else
							{
								cout << "新变量的标识符非法" << endl;
							}
						}
					}
					else
					{
						cout << "表达式非法" << endl;
					}
				}
				else
				{
					if (instructions.size() == 2) //fotmat: calc expression
					{
						bool isHave$ = false;
						for (unsigned int i = 0; i < instructions[1].size(); ++i)
						{
							if (instructions[1][i] == '

 
)
							{
								isHave$ = true;
							}
						}
						if (!isHave$)
						{
							Rational_Numbers result = calcEngine(instructions[1]);
							if (!wrong)//确认表达式计算正确
							{
								cout << "结果是：" << result << "   实数结果为" << result.realNumber() << endl;
							}
							else//计算过程出错就直接退出
							{
								break;
							}
						}
						else
						{
							cout << "表达式非法" << endl;
						}
					}
					else
					{
						cout << "输入正确数量的参数" << endl;
					}
				}
			
				break;
			//================================================//
			//=====================HELP=======================//
			//================================================//
			case HELP:
				if (instructions.size() == 1)
				{
					cout << "WRITE 1.输入参数创建新的/覆盖变量.       格式: write name numerator(分子) denominator(分母)" << endl;
					cout << "      2.用已经有的变量创建新的/覆盖变量. 格式: write name existedName" << endl << endl;

					cout << "DELETE 删掉已有的变量. 格式: write name existedName" << endl << endl;

					cout << "RENAME 更改变量的标识符. 格式: rename oldname newname" << endl << endl;

					cout << "LIST 输出所有变量. 格式:list" << endl << endl;

					cout << "READ 输出特定变量. 格式:read" << endl << endl;

					cout << "CALC 1.计算表达式并将结果创建新的/覆盖变量. 格式: calc name expression" << endl;
					cout << "     2.计算表达式并将结果输出.              格式: calc expression" << endl;
					cout << "     正确表达式示例:test1*-test2-2\*test3/233\*test4+666/-233" << endl;
					cout << "     注意表达式中间不能有空格, 且不能是test*0.1这样, 如果想要乘0.1等小数需要创建一个变量值为1/10然后计算" << endl << endl;;

					cout << "SAVE 将所有变量保存. 格式: save" << endl << endl;

					cout << "LOAD 将保存的变量载入程序. 格式: load" << endl << endl;

					cout << "EXIT 退出程序. 格式: exit" << endl << endl;

					cout << "PS:  所有指令以及变量不区分大小写" << endl;
				} 
				else
				{
					cout << "输入正确数量的参数" << endl;
				}

				break;
			//================================================//
			//=====================SAVE=======================//
			//================================================//
			case SAVE:
				if (instructions.size() == 1)
				{
					string temp;
					cout << "确定要这么做么, 这将导致之前的文件被覆盖(y/n)" << endl;
					getline(cin, temp);
					if (temp == "y")
					{
						fstream strm;
						strm.open(getDataDir(), ios::ate | ios::out);//覆盖方式打开, 若不存在此文件则创建
						for (unsigned int i = 0; i < nums.size(); ++i)
						{
							strm << nums[i].name << endl << nums[i].numerator << endl << nums[i].denominator << endl;
						}
						cout << "保存成功" << endl;
					}
					else
					{
						cout << "已取消" << endl;
					}
				}
				else
				{
					cout << "输入正确数量的参数" << endl;
				}

				break;
			//================================================//
			//=====================LOAD=======================//
			//================================================//
			case LOAD:
				if (instructions.size() == 1)
				{
					ifstream strm;
					strm.open(getDataDir());

					if (strm.good())
					{
						string strtemp;					
						Rational_Numbers objtemp;
						nums.clear();//删除当前所有的变量
						while (true)
						{
							getline(strm, strtemp);
							objtemp.name = strtemp;
							getline(strm, strtemp);
							objtemp.numerator = str2int(strtemp);
							getline(strm, strtemp);
							objtemp.denominator = str2int(strtemp);
							if (!strm.good())
							{
								break;//如果读取完毕, 退出循环.
							}
							nums.push_back(objtemp); //将变量录入程序
						}				
						cout << "读取成功" << endl;
					}
					else
					{
						cout << "没有找到储存文件" << endl;
					}
				}
				else
				{
					cout << "输入正确数量的参数" << endl;
				}

				break;
			//================================================//
			//====================RENAME======================//
			//================================================//
			case RENAME: //format: rename name newname
				if (instructions.size() == 3)
				{
					if (checkVaild(instructions[2])) //检查新名字是否合法
					{
						int add = getAddress(instructions[1]);
						if (add != -1)
						{
							nums[add].name = instructions[2];
							cout << "重命名成功" << endl; //本来还想加个新名字和旧名字一样不能改, 后来想想又要再加个if 而且没啥用...就算了
						}
						else
						{
							cout << "没有这个标识符的变量" << endl;
						}
					}
					else
					{
						cout<<"标识符非法" << endl;
					}		
				}
				else
				{
					cout << "输入正确数量的参数" << endl;
				}

				break;
			//================================================//
			//=====================LIST=======================//
			//================================================//
			case LIST: //format: list
				if (instructions.size() == 1) //检查参数数量是否正确
				{
					cout.setf(std::ios::left);
					cout.width(5);
					cout << "==============================================================================" << endl;
					cout << setw(8) << "编号" << setw(15) << "标识符" << setw(20) << "值" << "实数" << endl;
					for (unsigned int i = 0; i < nums.size(); ++i)
					{
						cout << setw(8) << i + 1 << setw(15) << nums[i].name << setw(20) << nums[i] << nums[i].realNumber() << endl; //重载过<<符号, 所以能直接输出
					}
					cout << "==============================================================================" << endl;
					cout.unsetf(std::ios::left);
				}
				else
				{
					cout << "输入正确数量的参数" << endl;
				}

				break;
			//================================================//
			//=====================READ=======================//
			//================================================//
			case READ:
				if (instructions.size() == 2)
				{
					int add = getAddress(instructions[1]);
					if (add != -1)
					{
						cout << instructions[1] << "的值为：" << nums[add] <<"  实数值为"<< nums[add].realNumber() << endl;
					}
					else
					{
						cout << "不存在标识符为" << instructions[1] << "的变量" << endl;
					}
				}
				else
				{
					cout << "输入正确数量的参数" << endl;
				}
				break;
			}

		}
	}		
};

int main() 
{
	MyioEngine ioEngine;
	ioEngine.start();

	system("pause");
    return 0;
}
```