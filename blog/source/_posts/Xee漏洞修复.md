title: Xee漏洞修复

date: 2016-05-28

categories: [web安全] 

---
> 在使用xml解析器解析xml文件或者是xml格式的字符串时,如果文件或字符串中带有外部实体会直接解析外部实体内的内容,有可能造成恶意入侵。

<!--more-->

## Xee漏洞  XML External Entity
 
    使用dom4j解析xml文件可能会引起外部实体的恶意注入

    比如xml文件如下：
```xml
<?xml version="1.0" encoding="utf-8"?>
        <!DOCTYPE entity [
                <!ENTITY xxe SYSTEM "file:///E:/study/base-utils-demo/src/main/resources/META-INF/spring/Service-bean.xml">
                <!ELEMENT entity ANY>
                <!ELEMENT x ANY>
                ]>
<entity>
<x>
    &xxe;
</x>
</entity>
```
当用下面的代码解析该xml文件时,会将本地的文件一并读取,这样如果外部恶意读取本地的文件(密码等敏感信息)会变得非常容易。
```java
XmlParse xmlParse = new XmlParse();
String text = xmlParse.generateResponseBody();

DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setXIncludeAware(false);
dbf.setExpandEntityReferences(false);

//获取具体的dom解析器
DocumentBuilder db = dbf.newDocumentBuilder();

// 解析xml文档 转换为dom4j
org.w3c.dom.Document documentW3c = db.parse(new InputSource(new StringReader(text)));
DOMReader domReader = new DOMReader();
Document document = domReader.read(documentW3c);
String text1 = document.asXML();
System.out.println(text1);
```
xml格式的字符串如下：

```java
    public String generateResponseBody() {
        StringBuilder sb = new StringBuilder();
        String resp = null;
        sb.append("<?xml version=\"1.0\" encoding=\"utf-8\"?>").append("\n");
        sb.append(" <!DOCTYPE entity [").append("\n");
        sb.append("<!ENTITY xxe SYSTEM \"file:///E:/study/base-utils-demo/src/main/resources/META-INF/spring/Service-bean.xml\">").append("\n");
        sb.append("<!ELEMENT entity ANY>").append("\n");
        sb.append("<!ELEMENT x ANY>").append("\n");
        sb.append("]>").append("\n");
        sb.append("<entity>").append("\n");
        sb.append("<x>").append("\n");
        sb.append("&xxe;").append("\n");
        sb.append("</x>").append("\n");
        sb.append("</entity>").append("\n");

        return sb.toString();

    }
```
当没有加防护时，解析出的xml文件如下：
```

<?xml version="1.0" encoding="utf-8"?>
 <!DOCTYPE entity [
<!ENTITY xxe SYSTEM "file:///E:/study/base-utils-demo/src/main/resources/META-INF/spring/Service-bean.xml">
<!ELEMENT entity ANY>
<!ELEMENT x ANY>
]>
<entity>
<x>
&xxe;
</x>
</entity>

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE entity><entity>
<x>

<beans xmlns="http://www.springframework.org/schema/beans" xmlns:context="http://www.springframework.org/schema/context" xmlns:dubbo="http://code.alibabatech.com/schema/dubbo" xmlns:tx="http://www.springframework.org/schema/tx" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" default-autowire="byName" xmlns:xml="" xml:xml:base="file:///E:/study/base-utils-demo/src/main/resources/META-INF/spring/Service-bean.xml" xmlns:xsi="" xsi:xsi:schemaLocation="http://www.springframework.org/schema/beans         http://www.springframework.org/schema/beans/spring-beans.xsd         http://www.springframework.org/schema/tx         http://www.springframework.org/schema/tx/spring-tx-3.0.xsd         http://www.springframework.org/schema/context         http://www.springframework.org/schema/context/spring-context-3.0.xsd         http://code.alibabatech.com/schema/dubbo         http://code.alibabatech.com/schema/dubbo/dubbo.xsd         ">


    <context:context:component-scan xmlns:context="" base-package="com.common.base.utils.SpringUtils"/>
        <bean xmlns="" class="com.common.base.utils.SpringUtils.SpringBeanUtil" id="SpringBeanUtil"/>
        <bean xmlns="" class="com.common.base.utils.SpringUtils.ThreadRunner" id="threadRunner">
                <property name="serviceBean" ref="serviceBean"/>
        </bean>
</beans>

</x>
</entity>
```
会将外部实体的内容解析出来,这样造成本地资源文件泄露。

## Xee漏洞解决办法

 * 配置解析器如下：
 ```
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
FEATURE = "http://xml.org/sax/features/external-general-entities";
dbf.setFeature(FEATURE, false);
FEATURE = "http://xml.org/sax/features/external-parameter-entities";
dbf.setFeature(FEATURE, false);
dbf.setXIncludeAware(false);
dbf.setExpandEntityReferences(false);

//获取具体的dom解析器
DocumentBuilder db = dbf.newDocumentBuilder();
 ```
 解析Xml格式的字符串
 
 ```
 // 解析xml文档 转换为dom4j
org.w3c.dom.Document documentW3c = db.parse(new InputSource(new StringReader(text)));
DOMReader domReader = new DOMReader();
Document document = domReader.read(documentW3c);
String text1 = document.asXML();
System.out.println(text1);
```
 
解析后xml文件如下：

```
<?xml version="1.0" encoding="utf-8"?>
 <!DOCTYPE entity [
<!ENTITY xxe SYSTEM "file:///E:/study/base-utils-demo/src/main/resources/META-INF/spring/Service-bean.xml">
<!ELEMENT entity ANY>
<!ELEMENT x ANY>
]>
<entity>
<x>
&xxe;
</x>
</entity>

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE entity><entity>
<x>

</x>
</entity>
```

并没有解析外部实体内容,问题解决。
