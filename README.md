# ZKTest

    public class DistributedSystemClient {

        private volatile List<String> servers = null;
        private ZooKeeper zk = null;

        // 获取zk连接
        private void getZkClient() throws Exception {

            // 服务器在需求中并不需要做任何监听
            zk = new ZooKeeper(GlobalConstants.zkhosts,
                    GlobalConstants.sessionTimeout, new Watcher() {

                @Override
                public void process(WatchedEvent event) {

                    if (event.getType() == EventType.None)
                        return;

                    try {
                        // 获取新的服务器列表,重新注册监听
                        updateServers();

                    } catch (Exception e) {

                        e.printStackTrace();
                    }

                }
            });

        }

        /**
         * 从zk中获取在线服务器信息
         */
        public void updateServers() throws Exception {

            // 从servers父节点下获取到所有子节点，并注册监听
            List<String> children = zk.getChildren(GlobalConstants.parentZnodePath,
                    true);

            ArrayList<String> serverList = new ArrayList<String>();

            for (String child : children) {

                byte[] data = zk.getData(GlobalConstants.parentZnodePath + "/"
                        + child, false, null);

                serverList.add(new String(data));

            }

            // 如果客户端是一个多线程程序，而且各个线程都会竞争访问servers列表，所以，在成员中用volatile修饰了一个servers变量
            // 而在更新服务器信息的这个方法中，是用一个临时List变量来进行更新
            servers = serverList;

            // 将更新之后的服务器列表信息打印在控制台观察一下
            for (String server : serverList) {

                System.out.println(server);
            }
            System.out.println("===================");

        }

        /**
         * 业务逻辑
         *
         * @throws InterruptedException
         */
        private void requestService() throws InterruptedException {
            System.out.println("I am cousumer");

        }

        public static void main(String[] args) throws Exception {

            DistributedSystemClient client = new DistributedSystemClient();

            // 先构造一个zk的连接
            client.getZkClient();

            // 获取服务器列表
            client.updateServers();

            // 客户端进入业务流程，请求服务器的服务
            client.requestService();

        }
    }
public class DistributedSystemServer {

    private ZooKeeper zk = null;

    private void getZkClient() throws Exception {

        // 服务器在需求中并不需要做任何监听
        zk = new ZooKeeper(GlobalConstants.zkhosts,
                GlobalConstants.sessionTimeout, null);

    }

    /**
     * 向zookeeper中的/servers下创建子节点
     *
     * @throws InterruptedException
     * @throws KeeperException
     */
    private void connectZK(String serverName, String port) throws Exception {

        // 先创建出父节点
        if (zk.exists(GlobalConstants.parentZnodePath, false) == null) {
            zk.create(GlobalConstants.parentZnodePath, null,
                    Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        }

        // 连接zk创建znode
        zk.create(GlobalConstants.parentZnodePath + "/",
                (serverName + ":" + port).getBytes(), Ids.OPEN_ACL_UNSAFE,
                CreateMode.EPHEMERAL_SEQUENTIAL);
        System.out.println("server " + serverName + " is online ......");

    }

    // 服务器的具体业务处理功能
    private void handle(String serverName) throws Exception {
        System.out.println("server " + serverName
                + " is waiting for task process......");
        Thread.sleep(Long.MAX_VALUE);

    }

    public static void main(String[] args) throws Exception {

        DistributedSystemServer server = new DistributedSystemServer();

        // 获取与zookeeper通信的客户端连接
        server.getZkClient();

        // 一启动就去zookeeper上注册服务器信息，参数1： 服务器的主机名 参数2：服务器的监听端口
        server.connectZK("127.0.0.1", "10001");

        // 进入业务逻辑处理流程,这里的意义只是为了不让它马上执行完毕而是让他一直休眠等待作为服务
        server.handle("127.0.0.1");

    }

}
