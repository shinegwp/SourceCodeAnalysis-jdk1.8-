## ClassLoader源码简单分析
[toc]
### loadClass方法

```
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException{
        
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);//从已经加载的类中寻找
            if (c == null) {
                long t0 = System.nanoTime();
                try {//双亲委派模型
                    if (parent != null) {//若父类不为null，则通过父类加载
                        c = parent.loadClass(name, false);
                    } else {//若父类为null，说明父类是bootstrapClassloader
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {//若父类没有进行加载，则采用自定义的方法加载
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);//默认的null方法，需要子类实现

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```
[toc]
### 自己手写的类加载器

```
public class MyClassLoader extends ClassLoader{
	private String path;
	private String name;
	public MyClassLoader(String name, String path) {
		super();//让系统类加载器成为该类的父加载器
		this.name = name;
		this.path = path;
	}
	public MyClassLoader(ClassLoader parent, String name, String path) {
		super(parent);//显示指定父类加载器
		this.name = name;
		this.path = path;
	}
	@Override
	protected Class<?> findClass(String name) throws ClassNotFoundException {
		//获取字节数组
		byte[] data = readClassFileToByteArray(name);
		return this.defineClass(name, data, 0, data.length);
	}
	private byte[] readClassFileToByteArray(String name2) {
		InputStream is = null;//写入
		byte[] returnData = null;
		name = name.replaceAll("\\.", "/");
		String filePath = this.path + name + ".class";
		File file = new File(filePath);
		ByteArrayOutputStream os = null;
		try {
			is = new FileInputStream(file);
			int tmp = 0;
			while ((tmp = is.read()) != -1) {
				os.write(tmp);
			}
			returnData = os.toByteArray();
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			try {
				if (os != null) {
					os.close();
				}
				if (is != null) {
					is.close();
				}
			} catch (IOException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
		}
		return returnData;
	}
}


```
