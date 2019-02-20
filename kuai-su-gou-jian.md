---
description: 如何快速搭建和开发pk类小游戏服务器
---

# 快速构建

针对不同的游戏项目，只需要copy一份新的项目工程，把Jelly-service中的代码重写即可

![](.gitbook/assets/image%20%281%29.png)



1.首先添加一个协议入口，如进入游戏 WebSocketHolder.ENTER\_ROOM，调用对应的业务实现类

2.新建业务逻辑类，具体的游戏处理逻辑

3.数据持久化，通过把数据写入对应的数据库或者缓存中（操作数据和普通项目一致，采用Mybatis框架，Jelly-dao子项目下）

```
/**
```

```text
 * 业务分发器.
 *
 */
public class Dispatcher {

    /**
    *  游戏入口
    * @param bt
     */
    public static void dispatch(RequestVo bt) {
       try {
          BaseRequest roomRequest = BaseRequest.parseFrom(bt.getBt());
          String jsonFormat =JsonFormat.printToString(roomRequest);
          //System.out.println(jsonFormat);                       //JSON格式
          JSONObject json = JSONObject.parseObject(jsonFormat);
          WebSocketHolder socketHolder = new WebSocketHolder();
          String data = json.getString("data");
          String time = json.getString("timestamp");
          if(time == null){
             time = String.valueOf(new Date().getTime());
          }
          String action = json.getString("action");
          String from = json.getString("from");
          String roomId = json.getString("roomid");
          socketHolder.setAction(action);
          socketHolder.setChannel(bt.getChannel());
          socketHolder.setFrom(from);
          socketHolder.setData(data);
          socketHolder.setRoomId(roomId);
          if(socketHolder.getAction() != null){
               if(socketHolder.getAction().equals(WebSocketHolder.ENTER_ROOM)){  //进入房间
                  LoginService.getInstance().enterRoom(socketHolder,time);
                  return;
               }
               if(socketHolder.getAction().equals(WebSocketHolder.PLAY_GAME)){  //玩家游戏
                  GameService.getInstance().gamePlay(socketHolder,time);
                  return;
               }
               if(socketHolder.getAction().equals(WebSocketHolder.GAME_OVER)){ //结束游戏
                  GameService.getInstance().gameOver(socketHolder,time);
                  return;
               }
               if(socketHolder.getAction().equals(WebSocketHolder.GET_ROOM)){ //房间信息
                  LoginService.getInstance().getRoom(socketHolder,time);
                  return;
               }
               if(socketHolder.getAction().equals(WebSocketHolder.READY)){ //玩家准备
                  GameService.getInstance().ready(socketHolder,time);
                  return;
               }
            }else{
               String result = ResonpseUtil.errorResonpse();   //错误返回
               ResonpseUtil.response(bt.getChannel(), socketHolder,result);
            }
      } catch (InvalidProtocolBufferException e) {
         e.printStackTrace();
      }
    }
    
}
```



