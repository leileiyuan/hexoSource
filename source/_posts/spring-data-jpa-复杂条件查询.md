---
title: spring data jpa 复杂条件查询
date: 2016-10-29 00:28:21
tags:
 - spring
 - spring data jpa
categories: 
 - spring dta jpa
 - spring
---

使用 `spring data jpa` 封装的任意条件查询，可使用`Pageable`定义分页和排序。

### 一般方式
定义接口，继承`JpaRepository`和`JpaSpecificationExecutor`。dao支持可以继承`JpaRepository`，条件查询可以继承`JpaSpecificationExecutor`。

```java
@(spring data jpa)Repository("resultInfoReposintory")
public interface ResultInfoReposintory extends JpaRepository<ResultInfo,Long>,JpaSpecificationExecutor<ResultInfo>
```

`JpaSpecificationExecutor`接口其中的两个方法

```java
List<T> findAll(Specification<T> spec);	// 条件查询，不分页
Page<T> findAll(Specification<T> spec, Pageable pageable);	// 条件查询，分页和排序
```
代码实例

```java
// 构建Pageable对象，用于分页和排序
int page = 0;	  	  // 当前页
int pageSize = 20;	  // 每页显示多少条
// 排序条件，以创建时间，更新时间 降序
List<Order> orders = new ArrayList<Sort.Order>();
orders.add(new Order(Direction.DESC, "createTime"));	//createTime是ResultInfo实体中的属性名
orders.add(new Order(Direction.DESC, "modifyTime"));
Pageable pageable = new PageRequest(page,pageSize,new Sort(orders));

// 构建查询条件对象 specification
Specification<ResultInfo> specification = this.getSpecification(accNo, beginDate, endDate);

// 执行分页查询
Page<ResultInfo> page = resultInfoRepository.findAll(specification, pageable);
```

构建查询条件的方法。使用`CriteriaQuery`得到查询条件

```java
// 构建查询条件
// root.get("accNo")中的accNo和 root.get("transDate")中的transDate，都是实体类中的属性名
private Specification<ResultInfo> getSpecification(final String accNo, final String beginDate, final String endDate){
	Specification<ResultInfo> specification = new Specification<ResultInfo>() {
		@Override
		public Predicate toPredicate(Root<ResultInfo> root, CriteriaQuery<?> query, CriteriaBuilder cb) {
			// 存储 多个在查询条件
			List<Predicate> list = new ArrayList<Predicate>(); 
			
			list.add(cb.equal(root.get("accNo").as(String.class), accNo));
			
			Date begin = DateUtil.str2Date(beginDate, DateUtil.FORMAT_YYYY_MM_DD);
			Date end = DateUtil.str2Date(beginDate, DateUtil.FORMAT_YYYY_MM_DD);
			if(begin != null && end != null){
				// 交易日期
				list.add(cb.between(root.get("transDate").as(Date.class), begin, end));
			}
			Predicate[] p = new Predicate[list.size()];

				return query.where(list.toArray(p)).getRestriction();
		}
	};
	return specification;
}

```

也可以使用`CriteriaBuilder`得到查询条件。下面代码的`cb`即是`CriteriaBuilder`的实例对象

```java
Predicate[] p = new Predicate[list.size()];
return cb.and(list.toArray(p));
```

最后会得到一个`Page<T>`对象实例，保存数据和分页信息。
`Page<ResultInfo> page = resultInfoRepository.findAll(specification, pageable);`
Page接口中的方法

```java
int getNumber();			// 当前页，第一页是0
long getTotalElements();	// 总记录数
int getSize();				// 每页显示多少条
int getTotalPages();		// 总页数
boolean isLast();			// 是否是最后一页
boolean isFirst();			// 是否是第一页
int getNumberOfElements();	// 当前页的记录条数
List<T> getContent();		// 当前页的记录集合
```

### 自行封装

构建复杂的查询条件

```java
/** 
 * @Description 
 * @author leileiyuan
 * @date 2016年11月28日 上午10:11:26 
*/
@Service
public class EnterpriseServiceImpl{
	@Autowired
	private EnterpriseRepository enterpriseRepository;

	public Page<Enterprise> queryEnterpriseInfo(Enterprise enterprise) {
		// 构建Pageable对象，用于分页和排序
		int page = 0;	  	  // 当前页
		int pageSize = 20;	  // 每页显示多少条
		// 排序条件，以创建时间，更新时间 降序
		List<Order> orders = new ArrayList<Sort.Order>();
		orders.add(new Order(Direction.DESC, "createTime"));	//createTime是Enterprise实体中的属性名
		orders.add(new Order(Direction.DESC, "modifyTime"));
		Pageable pageable = new PageRequest(page,pageSize,new Sort(orders));
		
		// 构建查询条件，我们已经把请求的查询条件放在入参 enterprise中，
		Criteria<Enterprise> spec = this.getSpecifications(enterprise);
		
		Page<Enterprise> data = enterpriseRepository.findAll(spec, pageable);
		return data;
	}
	
	// 建构查询条件
	private Criteria<Enterprise> getSpecifications(Enterprise enterprise) {
		Criteria<Enterprise> criteria = new Criteria<Enterprise>();
		// 默认条件 
		// 企业状态为“正常”和“已到期”并且
		List<String> eStatus = new ArrayList<String>();
		eStatus.add(EnterpriseStatusEnum.normal.getValue());
		//eStatus.add(EnterpriseStatusEnum.overDue.getValue());
		criteria.add(Restrictions.in("eStatus", eStatus));							// 企业状态  为 正常和已到期
		// 审核状态为“风控审核通过”的
		List<String> chkStatus = new ArrayList<String>();
		chkStatus.add(OperationAuditStatusEnum.chkStatus4.getValue());
		chkStatus.add(enterprise.getChkStatus());									// 审核状态
		criteria.add(Restrictions.in("chkStatus", chkStatus));									
		
		criteria.add(Restrictions.like("eId", enterprise.getEId()));				// 企业编号
		criteria.add(Restrictions.like("eName", enterprise.getEName()));			// 企业名称
		// 入网开始时间 - 入网结束时间
		Date joinDateBegin = enterprise.getJoinDateBegin();
		Date joinDateEnd = enterprise.getJoinDateEnd();
		if(joinDateBegin!=null && joinDateEnd!=null){
			criteria.add(Restrictions.between("joinDate",joinDateBegin, joinDateEnd));
		}
		// 到期开始时间 - 到期结束时间
		Date eDueDateBegin = enterprise.geteDueDateBegin();
		Date eDueDateEnd = enterprise.geteDueDateEnd();
		if(eDueDateBegin!=null && eDueDateEnd!=null){
			criteria.add(Restrictions.between("eDueDate", eDueDateBegin,eDueDateEnd));
		}
		criteria.add(Restrictions.equal("eStatus", enterprise.getEStatus()));		// 企业状态
		
		return criteria;
	}
```

`Criteria`对象的封装。
实现了`Specification`接口，重写`public Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder builder)`方法。用来**存储所有的查询条件**。
下面的代码中的`toPredicate`用来**构建查询条件**。是`Criterion`接口中的`toPredicate`

 ```java
List<Predicate> predicates = new ArrayList<Predicate>();  
for(Criterion c : criterions){  
    predicates.add(c.toPredicate(root, query,builder));  
} 
 ```

`Criteria`对象完整代码

```java
import java.util.ArrayList;
import java.util.List;

import javax.persistence.criteria.CriteriaBuilder;
import javax.persistence.criteria.CriteriaQuery;
import javax.persistence.criteria.Predicate;
import javax.persistence.criteria.Root;

import org.springframework.data.jpa.domain.Specification;

/** 
 * Description: 查询条件的容器
 * 			Predicate 为拼接好的条件或条件组合
 *
 * @author leileiyuan
 * Create Date: 2016年11月28日 上午10:50:05
 * @param <T>
 */
public class Criteria<T> implements Specification<T> {
	
	private List<Criterion> criterions = new ArrayList<Criterion>();
	
	@Override
	public Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder builder) {
		 if (!criterions.isEmpty()) {  
	            List<Predicate> predicates = new ArrayList<Predicate>();  
	            for(Criterion c : criterions){  
	                predicates.add(c.toPredicate(root, query,builder));  
	            }  
	            // 将所有条件用 and 联合起来  
	            if (predicates.size() > 0) {  
	                return builder.and(predicates.toArray(new Predicate[predicates.size()]));  
	            }  
	        }  
		 	// 返回一个没有联接条件的Predicate
	        return builder.conjunction();  
	}
	
	/**
	 * Description: 增加简单的条件表达式
	 *
	 * @param criterion
	 * @author leileiyuan
	 * Create Date: 2015年6月25日 下午2:43:13
	 */
	public void add(Criterion criterion){  
        if(criterion!=null){  
            criterions.add(criterion);  
        }  
    }  
}
```


`Criterion`接口代码。定义条件表达式。
`toPredicate(...)`方法，跟`org.springframework.data.jpa.domain.Specification<T>`接口的`toPredicate`方法一样。用来**构建条件**的。

```java
package com.creditease.b2bsettle.basedata.util.page.criteria;

import javax.persistence.criteria.CriteriaBuilder;
import javax.persistence.criteria.CriteriaQuery;
import javax.persistence.criteria.Predicate;
import javax.persistence.criteria.Root;

/** 
 * Description: 条件表达式接口
 * 				Criteria是一种类型安全和更面向对象的查询。
 *
 * @author leileiyuan
 * Create Date: 2016年11月28日 上午10:51:07
 */
public interface Criterion {
	public enum Operator {  
        EQ, 		// 等于				equal
        NE, 		// 不等于				notEqual
        GT, 		// 大于				greaterThan
        LT, 		// 小于				lessThan
        GTE, 		// 大于等于			greaterThanOrEqualTo
        LTE, 		// 小于等于			lessThanOrEqualTo
        BETWEEN,	// 大于等于 并且 小于等于	between
        LIKE, 		// 模糊匹配			like
        OR,			// 或者				or
        AND			// 并且				and
    }
	
	/**
	 * Description: 代表一个条件或多个条件的组合
	 *
	 * @param root 
	 * 		代表Criteria查询的根对象，Criteria查询的查询根定义了实体类型，能为将来导航获得想要的结果
	 * @param query 
	 * 		CriteriaQuery 代表一个specific的顶层查询对象，包含查询的各个部分：如select 、from、where、group by、order by等
	 * @param builder 
	 * 		CriteriaBuilder 用来构造CriteriaQuery的构建对象
	 * @return Predicate 相当于条件或条件组合
	 * @author leileiyuan
	 * Create Date: 2015年6月25日 上午11:02:31
	 */
    public Predicate toPredicate(Root<?> root, CriteriaQuery<?> query,  
            CriteriaBuilder builder);  
}

```

`Criterion`接口的实现有两个，`LogicalExpression`和`SimpleExpression`。`LogicalExpression`用来构建逻辑表达式；`SimpleExpression`用来构建简单表达式

```java
import java.util.ArrayList;
import java.util.List;

import javax.persistence.criteria.CriteriaBuilder;
import javax.persistence.criteria.CriteriaQuery;
import javax.persistence.criteria.Predicate;
import javax.persistence.criteria.Root;

/** 
 * Description: 逻辑表达式 ,用于复杂条件的联接 包含 AND OR 
 * 			形如select * from tableName where (expression1) and (expression2)
 * 			and 用来连接expression1 和expression2 这里的and就是逻辑表达式
 *
 * @author leileiyuan
 * Create Date: 2016年11月28日 上午11:35:09
 */
public class LogicalExpression implements Criterion {
	private Criterion[] criterions;			// 逻辑表达式包含的表达式 
	private Operator operator;				// 运算符
	
	@Override
	public Predicate toPredicate(Root<?> root, CriteriaQuery<?> query, CriteriaBuilder builder) {
		
		// 存放 查询时要过滤的条件
		List<Predicate>  predicates = new ArrayList<Predicate>();
		for (Criterion criterion : criterions) {
			predicates.add(criterion.toPredicate(root, query, builder));
		}
		
		switch (operator) {
		case AND:
			return builder.and(predicates.toArray(new Predicate[predicates.size()]));
		case OR:
			return builder.or(predicates.toArray(new Predicate[predicates.size()]));
		default:
			return null;
		}
		
	}
	
	public LogicalExpression() {
	}

	public LogicalExpression(Criterion[] criterions, Operator operator) {
		super();
		this.criterions = criterions;
		this.operator = operator;
	}

}

```

```java
import javax.persistence.criteria.CriteriaBuilder;
import javax.persistence.criteria.CriteriaQuery;
import javax.persistence.criteria.Expression;
import javax.persistence.criteria.Path;
import javax.persistence.criteria.Predicate;
import javax.persistence.criteria.Root;

/** 
 * Description: 简单表达式。包括 EQ NE GT LT GTE LTE BETWEEN LIKE
 *
 * @author leileiyuan
 * Create Date: 2016年11月28日 上午11:16:58
 */
public class SimpleExpression implements Criterion {
	private String fieldName;			// 属性名
	private Object value;           	// 属性所对应的值  
	private Object value2;           	// 属性所对应的值 使用在between时，第二个比较的数据值
	private Operator operator;      	// 运算符

	/* (non-Javadoc)
	 * @see com.creditease.b2bsettle.basedata.util.criteria.Criterion#toPredicate(javax.persistence.criteria.Root, javax.persistence.criteria.CriteriaQuery, javax.persistence.criteria.CriteriaBuilder)
	 */
	@SuppressWarnings({ "rawtypes", "unchecked" }) 
	@Override
	public Predicate toPredicate(Root<?> root, CriteriaQuery<?> query, CriteriaBuilder builder) {
		// TODO Auto-generated method stub
		Path expression = root.get(fieldName);
		
		switch (operator) {
		case EQ:
			return builder.equal(expression, value);
		case NE:
			return builder.notEqual(expression, value);
		case GT:
			return builder.greaterThan(expression, (Comparable) value);
		case LT:
			return builder.lessThan(expression, (Comparable) value);
		case GTE:
			return builder.greaterThanOrEqualTo(expression, (Comparable) value);
		case LTE:
			return builder.lessThanOrEqualTo(expression, (Comparable) value);
		case BETWEEN:
			return builder.between(expression, (Comparable) value, (Comparable) value2);
		case LIKE:
			return builder.like((Expression<String>) expression, "%" + value +"%");
		default:
			return null;
		}
	}
	
	public SimpleExpression() {
	}
	
	public SimpleExpression(String fieldName, Object value, Operator operator) {
		super();
		this.fieldName = fieldName;
		this.value = value;
		this.operator = operator;
	}
	
	public SimpleExpression(String fieldName, Object value, Object value2, Operator operator) {
		super();
		this.fieldName = fieldName;
		this.value = value;
		this.value2 = value2;
		this.operator = operator;
	}

	public String getFieldName() {
		return fieldName;
	}

	public void setFieldName(String fieldName) {
		this.fieldName = fieldName;
	}

	public Object getValue() {
		return value;
	}

	public void setValue(Object value) {
		this.value = value;
	}

	public Object getValue2() {
		return value2;
	}

	public void setValue2(Object value2) {
		this.value2 = value2;
	}

	public Operator getOperator() {
		return operator;
	}

	public void setOperator(Operator operator) {
		this.operator = operator;
	}
	
}
```

逻辑表达式和简单表达式的条件构建

```java
import java.util.Collection;

import org.springframework.util.StringUtils;

import com.creditease.b2bsettle.basedata.util.page.criteria.Criterion.Operator;

/** 
 * Description: 条件构造器
 *
 * @author leileiyuan
 * Create Date: 2016年11月28日 下午1:55:57
 */
public class Restrictions {
	/**
	 * Description: 等于
	 *
	 * @param fieldName 属性名
	 * @param value		属性名对应的值
	 * @return 
	 * 		value为null返回null;
	 * 		value如果是java.lang.String类型的实例 转换成String值为""，也返回null
	 * @author leileiyuan
	 * Create Date: 2016年11月28日 下午1:59:35
	 */
	public static SimpleExpression equal(String fieldName, Object value) {
		if (StringUtils.isEmpty(value)) {
			return null;
		}
		return new SimpleExpression(fieldName, value, Operator.EQ);
	}
	
	/**
	 * Description: 不等于
	 *
	 * @param fieldName
	 * @param value
	 * @return
	 * @author leileiyuan
	 * Create Date: 2016年11月28日 下午2:04:17
	 */
	public static SimpleExpression notEqual(String fieldName, Object value) {
		if (StringUtils.isEmpty(value)) {
			return null;
		}
		return new SimpleExpression(fieldName, value, Operator.NE);
	}
	/**
	 * Description: 大于
	 *
	 * @param fieldName
	 * @param value
	 * @return
	 * @author leileiyuan
	 * Create Date: 2016年11月28日 下午2:04:44
	 */
	public static SimpleExpression greaterThan(String fieldName, Object value) {
		if (StringUtils.isEmpty(value)) {
			return null;
		}
		return new SimpleExpression(fieldName, value, Operator.GT);
	}
	/**
	 * Description: 小于
	 *
	 * @param fieldName
	 * @param value
	 * @return
	 * @author leileiyuan
	 * Create Date: 2016年11月28日 下午2:05:35
	 */
	public static SimpleExpression lessThan(String fieldName, Object value) {
		if (StringUtils.isEmpty(value)) {
			return null;
		}
		return new SimpleExpression(fieldName, value, Operator.LT);
	}
	/**
	 * Description: 大于等于
	 *
	 * @param fieldName
	 * @param value
	 * @return
	 * @author leileiyuan
	 * Create Date: 2016年11月28日 下午2:07:22
	 */
	public static SimpleExpression greaterThanOrEqualTo(String fieldName, Object value) {
		if (StringUtils.isEmpty(value)) {
			return null;
		}
		return new SimpleExpression(fieldName, value, Operator.GTE);
	}
	/**
	 * Description: 小于等于
	 *
	 * @param fieldName
	 * @param value
	 * @return
	 * @author leileiyuan
	 * Create Date: 2016年11月28日 下午2:07:57
	 */
	public static SimpleExpression lessThanOrEqualTo(String fieldName, Object value) {
		if (StringUtils.isEmpty(value)) {
			return null;
		}
		return new SimpleExpression(fieldName, value, Operator.LTE);
	}
	
	/**
	 * Description: 大于等于 并且 小于等于
	 *
	 * @param fieldName
	 * @param value
	 * @return
	 * @author leileiyuan
	 * Create Date: 2016年11月28日 下午2:08:43
	 */
	public static SimpleExpression between(String fieldName, Object value, Object value2) {
		if (StringUtils.isEmpty(value)) {
			return null;
		}
		return new SimpleExpression(fieldName, value, value2, Operator.BETWEEN);
	}
	/**
	 * Description: 模糊匹配
	 *
	 * @param fieldName
	 * @param value
	 * @param value2
	 * @return
	 * @author leileiyuan
	 * Create Date: 2016年11月28日 下午2:10:04
	 */
	public static SimpleExpression like(String fieldName, Object value) {
		if (StringUtils.isEmpty(value)) {
			return null;
		}
		return new SimpleExpression(fieldName, value, Operator.LIKE);
	}
	
	/**
	 * Description: 并且
	 *
	 * @param criterions
	 * @return
	 * @author leileiyuan
	 * Create Date: 2016年11月28日 下午2:12:31
	 */
	public static LogicalExpression and(Criterion...criterions){
		return new LogicalExpression(criterions, Operator.AND);
	}
	
	/**
	 * Description: 或者
	 *
	 * @param criterions
	 * @return
	 * @author leileiyuan
	 * Create Date: 2016年11月28日 下午2:13:06
	 */
	public static LogicalExpression or(Criterion...criterions){
		return new LogicalExpression(criterions, Operator.OR);
	}
	
	/**
	 * Description: 包含于
	 *
	 * @param fieldName
	 * @param values
	 * @return
	 * @author leileiyuan
	 * Create Date: 2016年11月28日 下午2:21:58
	 */
	@SuppressWarnings("rawtypes")
	public static LogicalExpression in(String fieldName, Collection values){
		SimpleExpression[] ses = new SimpleExpression[values.size()];
		
		int i = 0;
		for (Object value : values) {
			ses[i] = new SimpleExpression(fieldName, value, Operator.EQ);
			i++;
		}
		return new LogicalExpression(ses, Operator.OR);
	}
}

```

到此封装就基本完成了。

-----
`spring data jpa`还提供了更复杂的多表联接查询
```java
public class Qfjbxxdz {
    @Id
    @GeneratedValue(generator = "system-uuid")
    @GenericGenerator(name = "system-uuid", strategy = "uuid.hex")
    private String id;
    @OneToOne
    @JoinColumn(name = "qfjbxx")
    private Qfjbxx qfjbxx; //关联表
    private String fzcc;
    private String fzccName;
    @ManyToOne
    @JoinColumn(name = "criminalInfo")
    private CriminalInfo criminalInfo;//关联表
    @Column(length=800)
    private String bz;
    //get/set......
}
```

可以用类似的方法来调用。`qfjbxx`，`criminalInfo`是关联表的属性对象，`id`,`xm`分别是`Qfjbxx`和`CriminalInfo`实体类的属性

```java
Predicate p1 = cb.equal(root.join("qfjbxx").get("id").as(String.class), id);
Predicate p2 = cb.like(root.join("criminalInfo").get("xm").as(String.class), "%"+xm+"%");
```

`CriteriaQuery`对象也支持分组和排序

```java
// 设置groupBy的条件
query.groupBy(
    root.get("qid").as(String.class),
    root.get("fid").as(String.class)
);
```

```java
// 设置orderBy的条
query.orderBy(cb.asc(root.get("xm").as(String.class)));
```

