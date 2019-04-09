# TCP粘包拆包处理
TCP是可靠的吗？可能你马上会回答：**TCP是面向连接的，是可靠的**。这个回答可能出现在TCP和UDP的区别问题中回答最多了。

这句话是没错的。但是这句话的可靠是指TCP协议会保证传输的有序性，尽量不丢包。

但是因为TCP是无边界，也就是以流的形式在网络中传输的。而且因为需要为其保证上面说的可靠性，**最小单元（MTU）**，**Nagle算法**，**接收端应用层没及时读取接收缓冲区中的数据**等都会导致接收的TCP包是**不正常的**。

# 接收的消息有几种情况：
- 1.正常
![](https://i.imgur.com/pXUap0f.png)

- 2.拆包
![](https://i.imgur.com/HKJgUo7.png)

- 2.粘包
![](https://i.imgur.com/x2rtYfN.png)


# 算法实现

		/**
		 * 基于固定消息头的粘包拆包处理算法
		 */
		
		//接收缓存大小
		#define RECV_BUFF_SZIE 102400
		
		//接收缓存
		char g_szRecv[RECV_BUFF_SZIE];
		
		//命令类型
		enum CMD
		{
			CMD_LOGIN,
			CMD_ERROR
		};

		//消息头
		struct DataHeader
		{
			DataHeader()
			{
				dataLength = sizeof(DataHeader);
				cmd = CMD_ERROR;
			}
			short dataLength;
			short cmd;
		};
		
		//真正的数据消息
		struct DataLogin : DataHeader
		{
			DataLogin()
			{
				dataLength = sizeof(DataLogin);
				cmd = CMD_LOGIN;
			}
		
			char data[100];
		};
		
		/**
		 * 封装一个客户端fd数据类型
		 * socket fd+接收缓冲区
		 */
		class ClientSocket 
		{
		public:
			ClientSocket(int sockfd = -1):m_sockfd(sockfd), m_lastPos(0)
			{
				memset(m_szMsgBuf, 0, sizeof(m_szMsgBuf));
			}
		
		public:
			int sockfd()
			{
				return m_sockfd;
			}
		
			char* msgBuf()
			{
				return m_szMsgBuf;
			}
		
			int getLastPos()
			{
				return m_lastPos;
			}
			void setLastPos(int pos)
			{
				m_lastPos = pos;
			}
		
		private:
			int m_sockfd;	//socket fd
			char m_szMsgBuf[RECV_BUFF_SZIE * 10];	//每个客户端socket fd有单独的接收缓冲区
			int m_lastPos;	//当前接收缓冲区的最后位置
		};
		
		/**
		 * 接受数据函数
		 * @param  pClient 客户端结构体
		 * @return         0-正常  -1-异常
		 */
		int RecvData(ClientSocket* pClient)
		{
			//接收数据
			int nLen = (int)recv(pClient->sockfd(), g_szRecv, RECV_BUFF_SZIE, 0);
			if (nLen <= 0)
			{
				printf("recv error Socket=%d\n", pClient->sockfd());
				return -1;
			}
		
			//把接收到的数据拷贝到客户端缓存区，以便判断处理和不影响接收缓冲区
			memcpy(pClient->msgBuf() + pClient->getLastPos(), g_szRecv, nLen);
		
			//重设客户端接收缓存区指向位置
			pClient->setLastPos(pClient->getLastPos() + nLen);
		
			//只要接收缓冲区的长度大于消息头就一直处理数据（固定共4字节。数据长度变量[short]+命令变量[short]）
			while (pClient->getLastPos() >= sizeof(DataHeader))
			{
				//判断接收缓冲区的长度是否大于消息头附带的数据体的长度
				DataHeader* header = (DataHeader*)pClient->msgBuf();
				if (pClient->getLastPos() >= header->dataLength)
				{
					//到这里表明肯定是一个完整的数据包（消息头+数据体）
					//计算出接受缓冲区比对应接收类型（消息头附带了）的完整数据包多余的字节
					int nSize = pClient->getLastPos() - header->dataLength;
		
					//为了演示，这里仅仅是对命令做处理，没有对数据做处理
					OnNetMsg(pClient->sockfd(),header);
		
					//因已消费客户端缓冲区的一个完整数据包，把后面的数据往前挪对应数据包大小的位置
					memcpy(pClient->msgBuf(), pClient->msgBuf() + header->dataLength, nSize);
		
					//重设新的客户端接收缓冲区最后数据的位置
					pClient->setLastPos(nSize);
				}
				else 
				{
					break;
				}
			}
			return 0;
		}