# 		应用进程死亡后的Activity变化

​	应用进程在启动的过程中都会注册AppDeathRecipient，用于接收应用死亡时的消息，这个注册过程在AMS中：

```java
private final boolean attachApplicationLocked(IApplicationThread thread,int pid) {
    ......
    if (DEBUG_ALL) Slog.v(
                TAG, "Binding process pid " + pid + " to record " + app);
	final String processName = app.processName;
        try {
            AppDeathRecipient adr = new AppDeathRecipient(
                    app, pid, thread);
            thread.asBinder().linkToDeath(adr, 0);
            app.deathRecipient = adr;
        } catch (RemoteException e) {
            app.resetPackageList(mProcessStats);
            startProcessLocked(app, "link fail", processName);
            return false;
        }
    ......
}
```

​	这个AppDeathRecipient里面的处理方法

