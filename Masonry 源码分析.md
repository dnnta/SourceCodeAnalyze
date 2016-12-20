# Masonry 源码分析
`Masonry` 是基于 `AutoLayout` 的语法上的封装，相对 `AutoLayout` 的笨重，它写起来思绪流畅，读起来简单明了。

## View+MASAdditions
这个 `category` 是我们使用的主要入口，为 `UIView` 扩展了一堆属性、三个主要调用方法和一个查找最近公共 `superview` 的方法

```
@property (nonatomic, strong, readonly) MASViewAttribute *mas_left;
@property (nonatomic, strong, readonly) MASViewAttribute *mas_top;
....
```
其中申明的属性都是 `readonly`，返回一个 `MASViewAttribute` 的实例，用来封装当前视图（弱引用）和对应的 `NSLayoutAttribute`。这些属性主要用在后面拼装 `MASCompositeConstraint` 的  `childConstraints` 时使用。

```
// 主要调用入口
- (NSArray *)mas_makeConstraints:(void(^)(MASConstraintMaker *))block {
    self.translatesAutoresizingMaskIntoConstraints = NO;
    MASConstraintMaker *constraintMaker = [[MASConstraintMaker alloc] initWithView:self];
    block(constraintMaker);
    return [constraintMaker install];
}
```
这里直接初始化一个 `MASConstraintMaker` 的实例变量，并当做参数传给 `block` 执行，然后再调用这个实例的 `install` 方法给当前视图添加约束。

由于 `constraintMaker` 是个局部变量，调用结束就销毁，所以这里并没有 `retain cycle` 这种隐患。

 `mas_updateConstraints:` 和 `mas_remakeConstraints:` 这两个方法实现类似，只是额外设置了对应的修改和删除属性。

##MASConstraintMaker
这个类提供了工厂方法来返回约束，这些约束包括一一对应的约束，比如 `top`, `right` 等，还有组合约束，比如 `edges`, `size`, `center`。所有这些都是通过定义为只读属性来返回一个 `MASConstraint` 的子类的

### 单一约束
在 `make` 调用 `top`,`left`, `height` 等属性时内部执行的是

```
- (MASConstraint *)constraint:(MASConstraint *)constraint addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
  MASViewAttribute *viewAttribute = [[MASViewAttribute alloc] initWithView:self.view layoutAttribute:layoutAttribute];
  MASViewConstraint *newConstraint = [[MASViewConstraint alloc] initWithFirstViewAttribute:viewAttribute];
  // 下面代码逻辑经过精简
  newConstraint.delegate = self;
  [self.constraints addObject:newConstraint];
  return newConstraint;
}
```
此时由于传入的 `constraint` 全部都是`nil`, 最终生成一个 `MASViewConstraint` 的实例变量, 并设置代理，并且添加到可变数组后返回这个对象，进行链式操作。

### 组合约束
`edges`, `size`, `center`这几个属性调后直接执行 

```
- (MASConstraint *)addConstraintWithAttributes:(MASAttribute)attrs {
   ....
    // 上面省略部分代码， 主要用位操作来判断参数有没有传，如果传了就转化成 NSMutableArray<MASViewAttribute>
    NSMutableArray *children = [NSMutableArray arrayWithCapacity:attributes.count];
    
    for (MASViewAttribute *a in attributes) {
        [children addObject:[[MASViewConstraint alloc] initWithFirstViewAttribute:a]];
    }
    
    MASCompositeConstraint *constraint = [[MASCompositeConstraint alloc] initWithChildren:children];
    constraint.delegate = self;
    [self.constraints addObject:constraint];
    return constraint;
}
``` 
最终生成一个 `MASCompositeConstraint` 的实例变量，这个实例会一次性初始化 `attrs` 里定义的所有单一约束，并添加到内部的数组属性，设置代理为它自己，然后设置本身的代理为 `maker`，添加到数组后返回实例对象。

### attributes
这个属性是一个`block`，主要用来自由组合 `top`, `left`等单一的约束属性，`block`内部还是调用 `addConstraintWithAttributes:`, 执行后返回一个 `MASCompositeConstraint` 的实例变量

### install
调用后，循环调用内部数组里保存的 `MASConstraint` 子类的 `install` 方法，最终将约束添加到对应的视图上面

## MASConstraint
这是一个抽象类，是 `Masonary` 实现链式语法的核心。在这个类里面定义了很多返回一个 `block` 的方法，而这个返回的 `block` 执行后返回的仍然是 `MASConstraint` 对象，因此可以一直链下去。

链式语法中的点 `.` ，方法调用一般是使用 `[]`，当使用 `.` 调用时，一般先去找有没有对应的属性 `getter` 方法，或者是直接定义的方法。 `block` 执行是 `()`

下面分析链式语法执行过程

```
make.left.top.right.equalTo(self.superview);
```
### make.left
上面已经分析过，这里将返回一个 `MASViewConstraint` 实例

### make.left.top
实际上执行的 `MASConstraint` 类的 `top` 方法，调用过程如下

```
// MASConstraint.m
// make.left.top
- (MASConstraint *)top {
	 // 这个方法调用子类实现
    return [self addConstraintWithLayoutAttribute:NSLayoutAttributeTop];
}

// MASViewConstraint.m 
- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
   // 这些链式操作必须在 setLayoutRelation 之前执行
    NSAssert(!self.hasLayoutRelation, @"Attributes should be chained before defining the constraint relation");
	// 调用 delegate 也就是 maker 里的方法
    return [self.delegate constraint:self addConstraintWithLayoutAttribute:layoutAttribute];
}

// MASConstraintMaker.h
 - (MASConstraint *)constraint:(MASConstraint *)constraint addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    MASViewAttribute *viewAttribute = [[MASViewAttribute alloc] initWithView:self.view layoutAttribute:layoutAttribute];
    MASViewConstraint *newConstraint = [[MASViewConstraint alloc] initWithFirstViewAttribute:viewAttribute];
  // 这里 constraint 不为 nil, 并且是 MASViewConstraint 类型，
  // 此时会把 A(传过来的 constraint) 和这里生成的
  // B(MASViewConstraint) 合并成一个 C(MASCompositeConstraint) 后，
  // 用 C 替换 children 里的 A, 并设置相关代理后返回 C
    if ([constraint isKindOfClass:MASViewConstraint.class]) {
        //replace with composite constraint
        NSArray *children = @[constraint, newConstraint];
        MASCompositeConstraint *compositeConstraint = [[MASCompositeConstraint alloc] initWithChildren:children];
        compositeConstraint.delegate = self;
        [self constraint:constraint shouldBeReplacedWithConstraint:compositeConstraint];
        return compositeConstraint;
    }
    if (!constraint) {
        newConstraint.delegate = self;
        [self.constraints addObject:newConstraint];
    }
    return newConstraint;
}
```
### make.left.top.right
这里执行的上面返回的 `MASCompositeConstraint` 对象的 `right` 方法，完成后返回 `self`，调用过程如下

```
- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    [self constraint:self addConstraintWithLayoutAttribute:layoutAttribute];
    return self;
}

- (MASConstraint *)constraint:(MASConstraint __unused *)constraint addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    id<MASConstraintDelegate> strongDelegate = self.delegate;
    // 调用 delegate (maker) 方法返回的结果直接添加到 children
    MASConstraint *newConstraint = [strongDelegate constraint:self addConstraintWithLayoutAttribute:layoutAttribute];
    newConstraint.delegate = self;
    [self.childConstraints addObject:newConstraint];
    return newConstraint;
}
```

### make.left.top.right.equalTo(self.superview);
在`MASConstraint.h` 文件中即定义了一个 `equalTo` 方法，也定义了一个 `equalTo` 的宏（这个宏必须在定义了`MAS_SHORTHAND_GLOBALS`后启用）。 

这个链式语法中应该是使用的宏，把传递的参数自动装箱后执行 `equalTo` 这个方法返回的 `block`

```
// MASConstraint.h
- (MASConstraint * (^)(id attr))equalTo;
- (MASConstraint * (^)(id attr))greaterThanOrEqualTo;
- (MASConstraint * (^)(id attr))lessThanOrEqualTo;

- (MASConstraint * (^)(id))equalTo {
    return ^id(id attribute) {
    		// 调用子类的对应方法
        return self.equalToWithRelation(attribute, NSLayoutRelationEqual);
    };
}

// 参数自动装箱操作
#define mas_equalTo(...)                 equalTo(MASBoxValue((__VA_ARGS__)))

#ifdef MAS_SHORTHAND_GLOBALS
#define equalTo(...)                     mas_equalTo(__VA_ARGS__)
#endif
```

```
// MASCompositeConstraint.m
- (MASConstraint * (^)(id, NSLayoutRelation))equalToWithRelation {
    return ^id(id attr, NSLayoutRelation relation) {
    // 遍历 children，调用每个 MASViewConstraint 的对应方法
        for (MASConstraint *constraint in self.childConstraints.copy) {
            constraint.equalToWithRelation(attr, relation);
        }
        return self;
    };
}

// MASViewConstraint.m
- (MASConstraint * (^)(id, NSLayoutRelation))equalToWithRelation {
    return ^id(id attribute, NSLayoutRelation relation) {
        if ([attribute isKindOfClass:NSArray.class]) {
            NSAssert(!self.hasLayoutRelation, @"Redefinition of constraint relation");
            NSMutableArray *children = NSMutableArray.new;
            for (id attr in attribute) {
                MASViewConstraint *viewConstraint = [self copy];
                viewConstraint.layoutRelation = relation;
                viewConstraint.secondViewAttribute = attr;
                [children addObject:viewConstraint];
            }
            MASCompositeConstraint *compositeConstraint = [[MASCompositeConstraint alloc] initWithChildren:children];
            compositeConstraint.delegate = self.delegate;
            [self.delegate constraint:self shouldBeReplacedWithConstraint:compositeConstraint];
            return compositeConstraint;
        } else {
            NSAssert(!self.hasLayoutRelation || self.layoutRelation == relation && [attribute isKindOfClass:NSValue.class], @"Redefinition of constraint relation");
            self.layoutRelation = relation;
            self.secondViewAttribute = attribute;
            return self;
        }
    };
}

```

### setSecondViewAttribute 
用来设置 `_secondViewAttribute`，在后面 `install`时设置 `NSLayoutContraint` 的 `secondItem` 的 `attribute`

```
- (void)setSecondViewAttribute:(id)secondViewAttribute {
    if ([secondViewAttribute isKindOfClass:NSValue.class]) {
        // 对应 equalTo(@10) 这种情况
        [self setLayoutConstantWithValue:secondViewAttribute];
    } else if ([secondViewAttribute isKindOfClass:MAS_VIEW.class]) {
    		// 对应 equalTo(superview) 这种情况
        _secondViewAttribute = [[MASViewAttribute alloc] initWithView:secondViewAttribute layoutAttribute:self.firstViewAttribute.layoutAttribute];
    } else if ([secondViewAttribute isKindOfClass:MASViewAttribute.class]) {
    		// 对应 equalTo(otherView.mas_left) 这种情况
        _secondViewAttribute = secondViewAttribute;
    } else {
        NSAssert(NO, @"attempting to add unsupported attribute: %@", secondViewAttribute);
    }
}
```

### install
经过如上面一样的的一系列链式语法调用后，基本完成了对当前视图的约束对象(`MASViewConstraint` 对象)的创建，但此时还没有正式添加到视图上。

首先初始化创建 `AutoLayout` 约束对象的相关属性

```
MAS_VIEW *firstLayoutItem = self.firstViewAttribute.item;
NSLayoutAttribute firstLayoutAttribute = self.firstViewAttribute.layoutAttribute;
MAS_VIEW *secondLayoutItem = self.secondViewAttribute.item;
NSLayoutAttribute secondLayoutAttribute = self.secondViewAttribute.layoutAttribute;
```

判断是不是 `size` 类型的属性，并且还没有 `secondAttribute` 时，添加对应的`second`相关属性

```
// 上面调用的`setSecondViewAttribute` 如果传进来的是 NSValue, 
// 比如 top.equalTo(@10) 是不会设置 `secondLayoutAttribute`的，此时默认为 superview
// 如果是 `size` 类型，此时 `secondItem` 应该为空
if (!self.firstViewAttribute.isSizeAttribute && !self.secondViewAttribute) {
   secondLayoutItem = self.firstViewAttribute.view.superview;
   secondLayoutAttribute = firstLayoutAttribute;
}
```

`MASLayoutConstraint` 是 `NSLayoutConstraint` 子类，只是添加了一个 `mas_key`, 方便出错时调试。

```
MASLayoutConstraint *layoutConstraint
   = [MASLayoutConstraint constraintWithItem:firstLayoutItem
                                   attribute:firstLayoutAttribute
                                   relatedBy:self.layoutRelation
                                      toItem:secondLayoutItem
                                   attribute:secondLayoutAttribute
                                  multiplier:self.layoutMultiplier
                                    constant:self.layoutConstant];
    
layoutConstraint.priority = self.layoutPriority;
layoutConstraint.mas_key = self.mas_key;
```

查找约束需要添加的视图对象，如果存在`secondViewAttribute.view`就找公共` superview`, 如果是` size` 属性，就添加到 `firstView`, 否则添加到 `firstView.superview`

```
if (self.secondViewAttribute.view) {
   MAS_VIEW *closestCommonSuperview = [self.firstViewAttribute.view mas_closestCommonSuperview:self.secondViewAttribute.view];
   NSAssert(closestCommonSuperview,
            @"couldn't find a common superview for %@ and %@",
            self.firstViewAttribute.view, self.secondViewAttribute.view);
   self.installedView = closestCommonSuperview;
} else if (self.firstViewAttribute.isSizeAttribute) {
   self.installedView = self.firstViewAttribute.view;
} else {
// 链式调用中如果没有执行 `equalTo` 等关系时
   self.installedView = self.firstViewAttribute.view.superview;
}
```

如果需要更新约束时， 首先查找相似的约束对象，存在就直接更新约束的`constant`, 否则添加约束到`installView` 上

```
MASLayoutConstraint *existingConstraint = nil;
if (self.updateExisting) {
   existingConstraint = [self layoutConstraintSimilarTo:layoutConstraint];
}
if (existingConstraint) {
   // just update the constant
   existingConstraint.constant = layoutConstraint.constant;
   self.layoutConstraint = existingConstraint;
} else {
   [self.installedView addConstraint:layoutConstraint];
   self.layoutConstraint = layoutConstraint;
   [firstLayoutItem.mas_installedConstraints addObject:self];
}
```
至此一个约束就真正的被添加到 `installView` 上面了。

## 其他
* 给 `view` 和 `MASContraint` 都可以设置别名 `key`
* 用字典映射各种约束属性和条件，调试输出更直接
* 查找两个或者多个 `view` 的最近公共 `superview`

