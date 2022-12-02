# 【ChatServer】15 - 客户端


简单实现一个基于控制台的客户端，功能比较简单，直接使用系统API和c++11的线程库（pthread底层），不用muduo库了，一个线程读消息，一个线程发送消息。

用户登录后在聊天主界面，可以输入命令进行操作，主线程也会使用一组回调函数来处理各个命令。


```cpp
/*
 *  @Filename : main.cc
 *  @Description : ChatClient
 *  @Datatime : 2022/12/02 17:00:39
 *  @Author : xushun
 */

#include "json.hpp"
#include "Group.hh"
#include "User.hh"
#include "public.hh"

#include <vector>
#include <chrono>
#include <iostream>
#include <string>
#include <unordered_map>
#include <functional>
#include <atomic>
#include <thread>

#include <unistd.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <semaphore.h>

using json = nlohmann::json;

// 当前系统登录用户
User gCurrentUser;
// 当前登录用户的好友列表
std::vector<User> gCurrentUserFriendList;
// 当前登录用户的群组列表
std::vector<Group> gCurrentUserGroupList;
// 控制主菜单页面程序
bool isMainMenuRunning = false;
// 用于读写线程通信
sem_t rwsem;
// 记录登录状态
std::atomic_bool isLogin{false};

// 接收线程
void readHandler(int clientFd);
// 获取系统时间
std::string getCurrentTime();
// 主聊天页面
void mainMenu(int clientFd);
// 显示当前用户基本信息
void showCurrentUserData();

// chat client   主线程发送数据 子线程接收数据
int main(int argc, char **argv)
{
    // 命令行参数需要指定服务器的ip和端口
    if (argc < 3)
    {
        std::cerr << "command invalid! example: ./chatclient 127.0.0.1 6000" << std::endl;
        exit(-1);
    }
    char *ip = argv[1];
    uint16_t port = atoi(argv[2]);
    // client socket
    int clientFd = socket(AF_INET, SOCK_STREAM, 0);
    if (-1 == clientFd)
    {
        std::cerr << "socket create error" << std::endl;
        exit(-1);
    }
    // server addr
    sockaddr_in serverAddr;
    memset(&serverAddr, 0, sizeof(sockaddr_in));
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(port);
    serverAddr.sin_addr.s_addr = inet_addr(ip);
    // connect server
    if (-1 == connect(clientFd, (sockaddr *)&serverAddr, sizeof(sockaddr_in)))
    {
        std::cerr << "connect server error" << std::endl;
        close(clientFd);
        exit(-1);
    }
    // 初始化读写线程通信用的信号量
    sem_init(&rwsem, 0, 0);
    // 启动子线程 接收服务器发来的数据
    std::thread readThread(readHandler, clientFd);
    readThread.detach();

    // 主线程用于接收用户输入 发送数据到服务器
    for (;;)
    {
        // show menu
        std::cout << "========================" << std::endl;
        std::cout << "1. login" << std::endl;
        std::cout << "2. register" << std::endl;
        std::cout << "3. quit" << std::endl;
        std::cout << "========================" << std::endl;
        std::cout << "choice:";
        // select function
        int choice = 0;
        std::cin >> choice;
        std::cin.get(); // 吃掉回车
        // 执行功能选项
        switch (choice)
        {
        case 1: // login
        {
            uint32_t id = 0;
            std::string password;
            std::cout << "userid: ";
            std::cin >> id;
            std::cin.get(); // 吃掉回车
            std::cout << "password: ";
            getline(std::cin, password);

            json js;
            js["msg_type"] = LOGIN_MSG;
            js["id"] = id;
            js["password"] = password;
            std::string request = js.dump();

            isLogin = false;
            int n = send(clientFd, request.c_str(), strlen(request.c_str()) + 1, 0);
            if (n == -1)
            {
                std::cerr << "send login message error: " << request << std::endl;
            }
            // 等待信号量 子线程处理完登录响应消息会通知
            sem_wait(&rwsem);
            if (isLogin)
            {
                // 登录成功 进入聊天主菜单
                isMainMenuRunning = true;
                mainMenu(clientFd);
            }
            break;
        }
        case 2: // register
        {
            std::string name;
            std::string password;
            std::cout << "username: ";
            std::getline(std::cin, name);
            std::cout << "password: ";
            std::getline(std::cin, password);

            json js;
            js["msg_type"] = REG_MSG;
            js["name"] = name;
            js["password"] = password;
            std::string request = js.dump();
            int n = send(clientFd, request.c_str(), strlen(request.c_str()) + 1, 0);
            if (n == -1)
            {
                std::cerr << "send register message error: " << request << std::endl;
            }
            // 等待信号量 子线程处理完注册响应消息会通知
            sem_wait(&rwsem);
            break;
        }
        case 3: // quit
        {
            close(clientFd);
            sem_destroy(&rwsem);
            exit(0);
        }
        default:
        {
            std::cerr << "invalid input!" << std::endl;
            break;
        }
        }
    }

    return 0;
}

// 处理注册响应的逻辑
void doRegResponse(json &responsejs)
{
    if (0 != responsejs["err_no"].get<int>())
    {
        // 注册失败
        std::cerr << "register fail: name already exist!" << std::endl;
    }
    else
    {
        // 注册成功
        std::cout << "register success, userid is: " << responsejs["id"] << ", donot forget it!" << std::endl;
    }
}

// 处理登录响应的逻辑
void doLoginResponse(json &responsejs)
{
    if (0 != responsejs["err_no"].get<int>())
    {
        // 登录失败 服务器会返回失败原因
        std::cerr << responsejs["err_msg"] << std::endl;
        isLogin = false;
    }
    else
    {
        // 登录成功
        // 记录当前用户
        gCurrentUser.setId(responsejs["id"].get<uint32_t>());
        gCurrentUser.setName(responsejs["name"]);
        // 记录当前用户好友列表
        if (responsejs.contains("friend_list"))
        {
            gCurrentUserFriendList.clear();
            std::vector<json> vecFrd = responsejs["friend_list"];
            for (json &ujs : vecFrd)
            {
                User user;
                user.setId(ujs["id"].get<uint32_t>());
                user.setName(ujs["name"]);
                user.setState(ujs["state"]);
                gCurrentUserFriendList.push_back(user);
            }
        }
        // 记录当前群组列表
        if (responsejs.contains("group_list"))
        {
            gCurrentUserGroupList.clear();
            std::vector<json> vecGrp = responsejs["group_list"];
            for (json &gjs : vecGrp)
            {
                Group group;
                group.setId(gjs["groupid"].get<uint32_t>());
                group.setName(gjs["groupname"]);
                group.setDesc(gjs["groupdesc"]);

                std::vector<json> vecUsr = gjs["users"];
                for (json &ujs : vecUsr)
                {
                    GroupUser user;
                    user.setId(ujs["id"].get<uint32_t>());
                    user.setName(ujs["name"]);
                    user.setState(ujs["state"]);
                    user.setRole(ujs["role"]);
                    group.getUsers().push_back(user);
                }
                gCurrentUserGroupList.push_back(group);
            }
        }
        // 显示当前用户的基本信息
        showCurrentUserData();
        // 显示当前用户的离线消息  个人消息和群组消息
        if (responsejs.contains("offline_msg"))
        {
            std::vector<std::string> vecMsg = responsejs["offline_msg"];
            for (std::string &msg : vecMsg)
            {
                json js = json::parse(msg);
                if (PEER_CHAT_MSG == js["msg_type"].get<int>())
                {
                    // 个人消息 time [id]name said: xxxx
                    std::cout << js["time"].get<std::string>() << " [" << js["id"] << "]"
                              << js["name"].get<std::string>() << " said: "
                              << js["msg_content"].get<std::string>() << std::endl;
                }
                else
                {
                    // 群组消息 time [Ggroupid][userid]username said: xxxx
                    std::cout << js["time"].get<std::string>() << " [G" << js["groupid"] << "]"
                              << "[" << js["userid"] << js["username"].get<std::string>()
                              << " said: " << js["msg_content"].get<std::string>() << std::endl;
                }
            }
        }
        isLogin = true;
    }
}

void readHandler(int clientFd)
{
    for (;;)
    {
        char buffer[4096] = {0};
        int n = recv(clientFd, buffer, 4096, 0); // 阻塞
        if (-1 == n || 0 == n)
        {
            close(clientFd);
            exit(-1);
        }
        // 接收chatserver的数据
        json js = json::parse(buffer);
        int msgType = js["msg_type"].get<int>();
        if (PEER_CHAT_MSG == msgType)
        {
            std::cout << js["time"].get<std::string>() << " [" << js["id"] << "]"
                      << js["name"].get<std::string>() << " said: "
                      << js["msg_content"].get<std::string>() << std::endl;
            continue;
        }
        if (GROUP_CHAT_MSG == msgType)
        {
            std::cout << js["time"].get<std::string>() << " [G" << js["groupid"] << "]"
                      << "[" << js["userid"] << js["username"].get<std::string>()
                      << " said: " << js["msg_content"].get<std::string>() << std::endl;
            continue;
        }
        if (LOGIN_ACK == msgType)
        {
            doLoginResponse(js);
            // 通知主线程 登录响应处理完成
            sem_post(&rwsem);
            continue;
        }
        if (REG_ACK == msgType)
        {
            doRegResponse(js);
            // 通知主线程 注册响应处理完成
            sem_post(&rwsem);
            continue;
        }
    }
}

void showCurrentUserData()
{
    std::cout << "======================login user======================" << std::endl;
    std::cout << "current login user => id:" << gCurrentUser.getId() << " name:" << gCurrentUser.getName() << std::endl;
    std::cout << "----------------------friend list---------------------" << std::endl;
    if (!gCurrentUserFriendList.empty())
    {
        for (User &user : gCurrentUserFriendList)
        {
            std::cout << user.getId() << " " << user.getName() << " " << user.getState() << std::endl;
        }
    }
    std::cout << "----------------------group list----------------------" << std::endl;
    if (!gCurrentUserGroupList.empty())
    {
        for (Group &group : gCurrentUserGroupList)
        {
            std::cout << group.getId() << " " << group.getName() << " " << group.getDesc() << std::endl;
            for (GroupUser &user : group.getUsers())
            {
                std::cout << user.getId() << " " << user.getName() << " " << user.getState()
                          << " " << user.getRole() << std::endl;
            }
        }
    }
    std::cout << "======================================================" << std::endl;
}

std::string getCurrentTime()
{
    auto tt = std::chrono::system_clock::to_time_t(std::chrono::system_clock::now());
    struct tm *ptm = localtime(&tt);
    char date[60] = {0};
    sprintf(date, "%d-%02d-%02d %02d:%02d:%02d",
            (int)ptm->tm_year + 1900, (int)ptm->tm_mon + 1, (int)ptm->tm_mday,
            (int)ptm->tm_hour, (int)ptm->tm_min, (int)ptm->tm_sec);
    return std::string(date);
}

// "help" command handler
void help(int fd = 0, std::string str = "");
// "chat" command handler
void peerchat(int, std::string);
// "addfriend" command handler
void addfriend(int, std::string);
// "creategroup" command handler
void creategroup(int, std::string);
// "joingroup" command handler
void joingroup(int, std::string);
// "groupchat" command handler
void groupchat(int, std::string);
// "logout" command handler
void logout(int, std::string);

// 客户端支持的命令列表
std::unordered_map<std::string, std::string> commandMap =
    {
        {"help", "       show all command,   fmt> help"},
        {"peerchat", "   peer to peer chat,  fmt> peerchat:{friendid}:{message content}"},
        {"addfriend", "  add a friend by id, fmt> addfriend:{friendid}"},
        {"creategroup", "create a group,     fmt> creategroup:{groupname}:{groupdesc}"},
        {"joingroup", "  join a group by id, fmt> joingroup:{groupid}"},
        {"groupchat", "  group chat,         fmt> groupchat:{groupid}:{message content}"},
        {"logout", "     log out,            fmt> logout"}};

// 客户端命令处理回调
std::unordered_map<std::string, std::function<void(int, std::string)>> commandHandlerMap =
    {
        {"help", help},
        {"peerchat", peerchat},
        {"addfriend", addfriend},
        {"creategroup", creategroup},
        {"joingroup", joingroup},
        {"groupchat", groupchat},
        {"logout", logout}};

void mainMenu(int clientFd)
{
    // 显示help菜单
    help();
    
    std::string buffer;
    while (isMainMenuRunning)
    {
        std::cout << gCurrentUser.getName() << "> ";
        std::getline(std::cin, buffer);
        // 存命令
        std::string command;
        int idx = buffer.find(":");
        if (-1 == idx)
        {
            command = buffer;
        }
        else
        {
            command = buffer.substr(0, idx);
        }
        auto itr = commandHandlerMap.find(command);
        if (itr == commandHandlerMap.end())
        {
            // 错误的命令
            std::cerr << "invalid input command!" << std::endl;
            continue;
        }
        // 调用命令对应的回调 mainMenu封闭 新功能无需修改
        itr->second(clientFd, buffer.substr(idx + 1, buffer.size() - idx));
    }
}

void help(int, std::string)
{
    std::cout << "command list >>>" << std::endl;
    for (auto& itr : commandMap)
    {
        std::cout << itr.first << " : " << itr.second << std::endl;
    }
    std::cout << std::endl;
}
void peerchat(int clientFd, std::string buffer)
{
    // {friendid}:{message content}
    int idx = buffer.find(":");
    if (-1 == idx)
    {
        std::cerr << "peerchat format invalid!" << std::endl;
        return;
    }
    int friendid = atoi(buffer.substr(0, idx).c_str());
    std::string message = buffer.substr(idx + 1, buffer.size() - idx);
    json js;
    js["msg_type"] = PEER_CHAT_MSG;
    js["id"] = gCurrentUser.getId();
    js["name"] = gCurrentUser.getName();
    js["to"] = friendid;
    js["msg_content"] = message;
    js["time"] = getCurrentTime();
    std::string request = js.dump();
    int n = send(clientFd, request.c_str(), strlen(request.c_str()) + 1, 0);
    if (-1 == n)
    {
        std::cerr << "send peerchat message error -> " << request << std::endl;
    }
}
void addfriend(int clientFd, std::string buffer)
{
    // {friendid}
    uint32_t friendid = atoi(buffer.c_str());
    json js;
    js["msg_type"] = ADD_FRIEND_MSG;
    js["userid"] = gCurrentUser.getId();
    js["friendid"] = friendid;
    std::string request = js.dump();
    int n = send(clientFd, request.c_str(), strlen(request.c_str()) + 1, 0);
    if (-1 == n)
    {
        std::cerr << "send addfriend message error -> " << request << std::endl;
    }
}
void creategroup(int clientFd, std::string buffer)
{
    // {groupname}:{groupdesc}
    int idx = buffer.find(":");
    if (-1 == idx)
    {
        std::cerr << "creategroup format invalid!" << std::endl;
        return;
    }
    std::string groupname = buffer.substr(0, idx);
    std::string groupdesc = buffer.substr(idx + 1, buffer.size() - idx);
    json js;
    js["msg_type"] = CREATE_GROUP_MSG;
    js["userid"] = gCurrentUser.getId();
    js["groupname"] = groupname;
    js["groupdesc"] = groupdesc;
    std::string request = js.dump();
    int n = send(clientFd, request.c_str(), strlen(request.c_str()) + 1, 0);
    if (-1 == n)
    {
        std::cerr << "send creategroup message error -> " << request << std::endl;
    }
}
void joingroup(int clientFd, std::string buffer)
{
    // {groupid}
    uint32_t groupid = atoi(buffer.c_str());
    json js;
    js["msg_type"] = JOIN_GROUP_MSG;
    js["userid"] = gCurrentUser.getId();
    js["groupid"] = groupid;
    std::string request = js.dump();
    int n = send(clientFd, request.c_str(), strlen(request.c_str()) + 1, 0);
    if (-1 == n)
    {
        std::cerr << "send joingroup message error -> " << request << std::endl;
    }
}
void groupchat(int clientFd, std::string buffer)
{
    // {groupid}:{message content}
    int idx = buffer.find(":");
    if (-1 == idx)
    {
        std::cerr << "groupchat format invalid!" << std::endl;
    }
    uint32_t groupid = atoi(buffer.substr(0, idx).c_str());
    std::string message = buffer.substr(idx + 1, buffer.size() - idx);
    json js;
    js["msg_type"] = GROUP_CHAT_MSG;
    js["userid"] = gCurrentUser.getId();
    js["username"] = gCurrentUser.getName();
    js["groupid"] = groupid;
    js["msg_content"] = message;
    js["time"] = getCurrentTime();
    std::string request = js.dump();
    int n = send(clientFd, request.c_str(), strlen(request.c_str()) + 1, 0);
    if (-1 == n)
    {
        std::cerr << "send groupchat message error -> " << request << std::endl;
    }
}
void logout(int clientFd, std::string buffer)
{
    json js;
    js["msg_type"] = LOGOUT_MSG;
    js["id"] = gCurrentUser.getId();
    std::string request = js.dump();
    int n = send(clientFd, request.c_str(), strlen(request.c_str()) + 1, 0);
    if (-1 == n)
    {
        std::cerr << "send logout message error -> " << request << std::endl;
    }
    else
    {
        isMainMenuRunning = false;
    }
}
```
