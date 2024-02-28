# @Autowired,@Component 구현하기
> 리플렉션으로 구현할 수 있는 어노테이션을 생각하던 중. 런타임시에 스프링이 빈을 주입하는 과정을 구현하면 좋겠다고 생각했다.


## 어노테이션 작성하기

### @Autowired
```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Autowired {
}
```
필드에 autowired가 붙어있을 경우 런타임시에 자동으로 주입해줘야 함으로 @Retention(RetentionPolicy.RUNTIME)`을 붙였다.

### @Component
```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Component {

}
```
@Target(ElementType.TYPE)의 경우 해당 어노테이션의 사용을 클래스로 정한다.

## 예제 작성하기
> 예제에서는 프린터기가 잉크를 필드로 가지고 있으며, 이를 정상적으로 주입받았는지 확인하는 것을 목적으로 했다.

### Printer
```java
@Component
class Printer {
    @Autowired
    private Ink ink;

    public void doSomething() {
        ink.performAction();
    }

    public Printer() {
        System.out.println("프린터기 작동...");
    }
}
```
### Ink
```java
@Component
class Ink {
    public void performAction() {
        System.out.println("파란색 잉크입니다!!");
    }

    public Ink() {
        System.out.println("잉크 카트리지 부착...");
    }
}
```
해당 잉크 클래스는 차후 인터페이스로 변경후에, 각 컬러잉크와 흑백잉크중 하나를 선택하는 것을 만드는 걸 목표로 한다.

### SimpleContainer
```java
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class SimpleContainer {
    private Map<Class<?>, Object> beans = new HashMap<>();

    public SimpleContainer() {
        try {
            initializeBeans();
            injectAllDependencies();
        } catch (Exception e) {
            throw new RuntimeException("컨테이너 초기화 중 에러 발생", e);
        }
    }

    private void initializeBeans() throws ReflectiveOperationException {

        List<Class<?>> classes = List.of(Printer.class, Ink.class);
        for (Class<?> clazz : classes) {
            if (clazz.isAnnotationPresent(Component.class)) {
                // 모든 컴포넌트 인스턴스화
                Object instance = clazz.getDeclaredConstructor().newInstance();
                beans.put(clazz, instance);
            }
        }
    }

    private void injectAllDependencies() throws IllegalAccessException {
        for (Object bean : beans.values()) {
            injectDependencies(bean);
        }
    }

    private void injectDependencies(Object instance) throws IllegalAccessException {
        for (Field field : instance.getClass().getDeclaredFields()) {
            if (field.isAnnotationPresent(Autowired.class)) {
                Object dependency = beans.get(field.getType());
                if (dependency != null) {
                    field.setAccessible(true);
                    field.set(instance, dependency);
                    System.out.println("의존성 주입: " + field.getType() + " -> " + instance.getClass());
                } else {
                    throw new IllegalStateException("의존성을 찾을 수 없습니다: " + field.getType());
                }
            }
        }
    }

    public <T> T getBean(Class<T> clazz) {
        return clazz.cast(beans.get(clazz));
    }
}
```
initializeBeans()를 통해 Component를 우선적으로 등록해준다
처음에는 @Component가 붙은 클래스가 등록될 때, 바로 필드를 주입해줬는데, 빈 등록 순서에 따라 오류를 발생시켰다.
따라서 모든 빈을 등록한 후에 주입을 해주는 것으로 변경했다.


## Main
```java

public class Main {
    public static void main(String[] args) {
        SimpleContainer container = new SimpleContainer();
        Printer printer = container.getBean(Printer.class);
        printer.doSomething();
    }
}
```
컨테이넌를 생성하고, 컨테이너에 등록된 빈을 가져와서 실행시켰다. 실행결과는 다음과 같다.
```text
프린터기 작동...
잉크 카트리지 부착...
의존성 주입: class Ink -> class Printer
파란색 잉크입니다!!
```

