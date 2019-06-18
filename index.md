## Description

In this program, a client is requesting data from a server over the network. Data request and retrieval is optimized by threading.  

### Main Server Function
In the main function for the server, the Port is determined by the user and the IP address will be localhost. The constructor for the Network Request Channel is called, which builds a vector of file descriptors for each client thread that connects to it. A thread is called for each connection between the client and server. The main function passes each thread to the handle_process_loop function where that server thread will wait for messages from the client until a quit message is receieved. When a quit message is receieved, the socket will return from the handle_process_loop function and join back to the main program. This main function will wait until all threads have joined, before it terminates. 
```
int main(int argc, char *argv[]) {
	string port;
	string IP;

	if (argc != 3) 
  {
		EXITONERROR("Server: incomplete command line arg");
	}

	port = string(argv[3]);
	IP = "127.0.0.1"; // localhost 
	NetworkRequestChannel control_channel("control", NetworkRequestChannel::SERVER_SIDE, IP, port);
  
  pthread_t* threads = new pthread_t[control_channel.getSockVec().size()];
  vector<NetworkRequestChannel*> net_channels;

	for (int i = 0; i < control_channel.getSockVec().size(); i++)
	{
		// declare a new request channel that holds the socket for each thread
		net_channels.push_back(new NetworkRequestChannel(control_channel.getSockVec().at(i)));
	}
	for (int i = 0; i < control_channel.getSockVec().size(); i++)
	{
		pthread_create(&threads[i], 0, handle_process_loop, net_channels.at(i));
	}

	// join threads
	int join_count = 0;
	for (int i = 0; i < control_channel.getSockVec().size(); i++)
	{
		join_count++;
		pthread_join(threads[i], 0);
		cout << "server thread " << join_count << " joined" << endl;
	}
	return 0;
  

}
```

## Sockets

Here is the Network Request Channel Constructor for the server side:

```
if (_side == SERVER_SIDE)
	{
		//set flags (localhost for server)
		hints.ai_flags = AI_PASSIVE;

		// get address info
		if ((status = getaddrinfo(NULL, port_ptr, &hints, &info)) != 0)
		{
			cerr << "getaddrinfo: " << gai_strerror(status) << endl;
			exit(1);
		}

		// create listener socket
		sockListen = socket(info->ai_family, info->ai_socktype, info->ai_protocol);
		if (sockListen < 0)
		{
			perror("server: socket");
			exit(1);
		}

		// bind socket
		if (bind(sockListen, info->ai_addr, info->ai_addrlen) == -1)
		{
			perror("server: bind");
			exit(1);
		}

		freeaddrinfo(info); // done with this struct

		// set a listener
		if (listen(sockListen, 10) == -1)
		{
			perror("server: listen");
			exit(1);
		}

		// connect initial contact with client
		if ((_socket_ = accept(sockListen, (struct sockaddr*)&client_addr, &client_sock_size)) == -1)
		{
			perror("server: ");
			exit(1);
		}

		// get # of worker threads from client
		char* buf = new char[MAX_MESSAGE];

		int byte_count;
		if ((byte_count = recv(_socket_, buf, MAX_MESSAGE, 0)) < 0)
		{
			perror("Error: receieve failure");
			exit(1);
		}
		datamsg* thread_msg = (datamsg*)buf;
		socket_vector.push_back(_socket_);
		cout << "socket " << _socket_ << " added to vector on server side" << endl;

		w = thread_msg->person;

		clients = w;  // set member variable

		// let them know we got it
		this->cwrite(buf, sizeof(datamsg) + 1);

		client_sock_size = sizeof(client_addr);
		for (int i = 0; i < w; i++)
		{
			// connect to client
			if ((_socket_ = accept(sockListen, (struct sockaddr*)&client_addr, &client_sock_size)) == -1)
			{
				perror("server: ");
				exit(1);
			}
			// add the socket fd to the vector
			socket_vector.push_back(_socket_);
			cout << "socket " << _socket_ << " added to vector on server side" << endl;
		}
	}
```

The getaddrinfo() function return a struct of connection info that is used by bind() and connect(). A listener socket is declared, to be used by bind() and accept(). Bind() assigns the Sockets file descriptor to the address info. Listen() sets the listener Socket to be passive, so that it can be used to receive incoming connections. Accept() is the function that connects to the client. A file descriptor is returned from accept() which is the communication channel between to connected parties. Next, the server receives its first message fromt the client (using recv()) which contains the number of threads that the client wants to connect. The server sends a message back to the client, confirming that the message has been receieved, then uses that number in the for loop which will establish all of the connections. In this loop, the server waits for a connection, and once received, puts the file descriptor for that socket into a vector which is used in the main function to create server threads. 

