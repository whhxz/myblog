---
title: Java设计模式-建造者模式（Builder Pattern）
date: 2017-09-14 15:42:00
categories: ['设计模式']
tags: ['设计模式', '建造者模式', 'Builder Pattern']
---
### 建造者模式
> 创建一个产品时，有时产品有不同的组成成分，在其组成成分没创建完成之前，改产品是不可使用的。使用建造者模式，通过创建所有产品内部零件，组装完成后返回给用户。

例子：
举例设计LOL角色，角色包含皮肤、技能、属性、性别
![](http://otxnth5wx.bkt.clouddn.com/20170914%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-09-14%E4%B8%8B%E5%8D%885.04.14.png)

Java实现：
```Java
/**
 * 角色
 */
class Actor{
    /**
     * 皮肤
     */
    private String skin;
    /**
     * 技能
     */
    private String skill;
    /**
     * 属性
     */
    private String attribute;
    /**
     * 性别
     */
    private String gender;

    public String getSkin() {
        return skin;
    }

    public void setSkin(String skin) {
        this.skin = skin;
    }

    public String getSkill() {
        return skill;
    }

    public void setSkill(String skill) {
        this.skill = skill;
    }

    public String getAttribute() {
        return attribute;
    }

    public void setAttribute(String attribute) {
        this.attribute = attribute;
    }

    public String getGender() {
        return gender;
    }

    public void setGender(String gender) {
        this.gender = gender;
    }

    @Override
    public String toString() {
        return "Actor{" +
                "skin='" + skin + '\'' +
                ", skill='" + skill + '\'' +
                ", attribute='" + attribute + '\'' +
                ", gender='" + gender + '\'' +
                '}';
    }
}
abstract class AbstractActorBuilder{
    protected Actor actor = new Actor();

    /**
     * 构建皮肤
     */
    public abstract void buildSkin();

    /**
     * 构建技能
     */
    public abstract void buildSkill();

    /**
     * 构建属性
     */
    public abstract void buildAttribute();

    /**
     * 构建性别
     */
    public abstract void buildGender();

    /**
     * 创建角色
     * @return
     */
    public Actor resultActor(){
        return actor;
    }

}

/**
 * 德玛西亚之力建造者
 */
class GarenActorBuilder extends AbstractActorBuilder{
    @Override
    public void buildSkin() {
        actor.setSkin("盖伦皮肤");
    }

    @Override
    public void buildSkill() {
        actor.setSkill("盖伦技能");
    }

    @Override
    public void buildAttribute() {
        actor.setAttribute("盖伦属性");
    }

    @Override
    public void buildGender() {
        actor.setGender("男");
    }
}

/**
 * 德邦总管建造者
 */
class XinZhaoActorBuilder extends AbstractActorBuilder{

    @Override
    public void buildSkin() {
        actor.setSkin("赵信皮肤");
    }

    @Override
    public void buildSkill() {
        actor.setSkill("赵信技能");
    }

    @Override
    public void buildAttribute() {
        actor.setAttribute("赵信属性");
    }

    @Override
    public void buildGender() {
        actor.setGender("男");
    }
}

/**
 * 寒冰射手
 */
class AsheActorBuilder extends AbstractActorBuilder{
    @Override
    public void buildSkin() {
        actor.setSkin("艾希皮肤");
    }

    @Override
    public void buildSkill() {
        actor.setSkill("艾希技能");
    }

    @Override
    public void buildAttribute() {
        actor.setAttribute("艾希属性");
    }

    @Override
    public void buildGender() {
        actor.setGender("女");
    }
}

/**
 * 建造组装
 */
class BuilderDirector{
    /**
     * 构建角色
     * @param actorBuilder
     * @return
     */
    public Actor constructActor(AbstractActorBuilder actorBuilder){
        actorBuilder.buildSkin();
        actorBuilder.buildSkill();
        actorBuilder.buildAttribute();
        actorBuilder.buildGender();
        return actorBuilder.resultActor();
    }
}

public class Main {
    public static void main(String[] args) {
        Actor asheActor = new BuilderDirector().constructActor(new AsheActorBuilder());
        //Actor{skin='艾希皮肤', skill='艾希技能', attribute='艾希属性', gender='女'}
        System.out.println(asheActor);
        Actor xinZhaoActor = new BuilderDirector().constructActor(new XinZhaoActorBuilder());
        //Actor{skin='赵信皮肤', skill='赵信技能', attribute='赵信属性', gender='男'}
        System.out.println(xinZhaoActor);
        Actor garenActor = new BuilderDirector().constructActor(new GarenActorBuilder());
        //Actor{skin='盖伦皮肤', skill='盖伦技能', attribute='盖伦属性', gender='男'}
        System.out.println(garenActor);

    }
}
```
在创建角色的时候，用户直接使用建造组装者创建创建相关角色，传入相关角色的创建者即可。在实际使用中，在创建各个组件的时候，应该是有规则的，通过不同的规则建造不同的角色。在后续如果新增了相关角色，可以新建一个角色建造者即可，无需修改其他相关代码。改处缺点在于，如果角色新增了其他相关组件，则修改较大。

### 建造者总结
建造者核心在于一步步构建一个包含多个组件的完整对象，与工厂模式不同在于通常建造者创建的对象相对工厂模式比较复杂，通常工厂模式创建的是单一或者相对不是很复杂的对象。如果将工厂模式看出汽车零件生产厂，建造者模式可以看出汽车装配厂。

**建造者优点：**
1. 用户不需要知道建造细节，将产品本身和产品创建进行解耦，使相同的创建过程可以创建不同的产品。
2. 扩展方便，符合"开闭原则"
3. 细化了产品创建，使创建过程清晰明了
**建造者缺点：**
1. 如果产品过多，会导致建造者过多，系统过于臃肿。
2. 如果产品组件过于复杂而且多变，会导致产品创建过程过于复杂。

**建造者使用范围：**
1. 需要生成的产品对象有复杂的内部结构，这些产品对象通常包含多个成员属性。
2. 需要生成的产品对象的属性相互依赖，需要指定其生成顺序。
3. 对象的创建过程独立于创建该对象的类。在建造者模式中通过引入了指挥者类，将创建过程封装在指挥者类中，而不在建造者类和客户类中。
4. 隔离复杂对象的创建和使用，并使得相同的创建过程可以创建不同的产品。
