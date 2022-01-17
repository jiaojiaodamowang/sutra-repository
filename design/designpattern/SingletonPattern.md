# Singleton Pattern

## 饥汉模式
```java
public class Singleton {

    private static final Singleton INSTANCE = new Singleton();

    private Singleton(){}

    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

## 双重校验模式
```java
public class Singleton {

    private volatile static Singleton instance;

    private Singleton(){}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

## 静态内部类模式
```java
public class Singleton {
    
    private static class SingletonHolder() {
        private static final Singleton INSTANCE = new Singleton();
    }

    private Singleton(){}
    
    public static Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```

## 枚举类模式
```java
public enum Singleton {
    
    INSTANCE;
    
    public Singleton getInstance() {
        return INSTANCE;
    }
}
```
