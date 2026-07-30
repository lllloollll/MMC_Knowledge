**声明式UI**

##### 布局基础知识

**基础布局：**

1. 列布局 `Column{}`

2. 行布局 `Row{}`

3. 盒子布局 `Box{}`

**如需在 `Row` 中设置子项的位置，请设置 `horizontalArrangement` 和 `verticalAlignment` 参数。对于 `Column`，请设置 `verticalArrangement` 和 `horizontalAlignment` 参数**

compose布局和view布局最大的区别：compose可以有效处理嵌套布局

**布局模型：**

Composition-->Layout-->Drawing

**布局的修饰符Modifier**

```kotlin
Column{
    Modifier
        .clickable(onclick = onClick)    //响应点击
        .padding(padding)                //设置padding
        .fillMaxWidth()                  //填充为父项最大宽度
        .fillMaxHeight()
        .size(width = 400.dp,height = 200.dp)                          //指定宽度和高度
        .sizeInt(minWidth,maxWidth,minHeight,maxHeight)
        .requiredSize()                    //优先级较高的指定尺寸，忽略父级的约束
        .offset(x = ,y = )                //偏移，不会改变其测量结果
        .absoluteOffset()
}


Text(
    Modifier
        .paddingFromBaseline(top = 50.dp)           //从布局顶部到文本基线保持特定距离
)


Image(
    painterResource(R.drawable.hero),
    contentDescription = null,
    Modifier
        .clip(CircleShape)
        .clip(RoundedCornerShape(8.dp))
        .padding(10.dp)
        .size(100.dp)
)
```

**自适应布局：** `BoxWithConstraints{}`

**基于槽位布局**

##### Compose修饰符

**作用：**

- 更改可组合项大小、布局、行为和外观

- 添加信息，如无障碍标签

- 处理用户输入

- 添加高级互动，如式元素可点击、可滚动、可拖动或可缩放

**修饰符顺序很重要**

**Compose中的作用域安全**

- Box：matchParentSize

- Row和Column：weight

**提取和重复使用修饰符**

```kotlin
// Now, the allocation of the modifier happens here:
val reusableModifier = Modifier
    .padding(12.dp)
    .background(Color.Gray)

@Composable
fun LoadingWheelAnimation() {
    val animatedState = animateFloatAsState(/*...*/)

    LoadingWheel(
        // No allocation, as we're just reusing the same instance
        modifier = reusableModifier,
        animatedState = animatedState
    )
}

Column(/*...*/) {
    val reusableItemModifier = Modifier
        .padding(bottom = 12.dp)
        // Align Modifier.Element requires a ColumnScope
        .align(Alignment.CenterHorizontally)
        .weight(1f)
    Text1(
        modifier = reusableItemModifier,
        // ...
    )
    Text2(
        modifier = reusableItemModifier
        // ...
    )
    // ...
}
```

##### 自定义修饰符

```kotlin
@Composable
fun Modifier.myBackground(): Modifier {
    val color = LocalContentColor.current
    return this then Modifier.background(color.copy(alpha = 0.5f))
}

@Composable
fun MyScreen() {
    CompositionLocalProvider(LocalContentColor provides Color.Green) {
        // Background modifier created with green background
        val backgroundModifier = Modifier.myBackground()

        // LocalContentColor updated to red
        CompositionLocalProvider(LocalContentColor provides Color.Red) {

            // Box will have green background, not red as expected.
            Box(modifier = backgroundModifier)
        }
    }
}
```

##### 修饰符列表

##### Compose中的分页器

- 左右水平滚动分页器：`HorizontalPager`
  
  ```kotlin
  // Display 10 items
  val pagerState = rememberPagerState(pageCount = {
      10
  })
  HorizontalPager(state = pagerState) { page ->
      // Our page content
      Text(
          text = "Page: $page",
          modifier = Modifier.fillMaxWidth()
      )
  }
  ```

- 上下滚动分页器：`VerticalPager`
  
  ```kotlin
  // Display 10 items
  val pagerState = rememberPagerState(pageCount = {
      10
  })
  VerticalPager(state = pagerState) { page ->
      // Our page content
      Text(
          text = "Page: $page",
          modifier = Modifier.fillMaxWidth()
      )
  }
  ```

##### Compose中的协程作用域

- `GlobalScope` 与组合生命周期无关，页面旋转、导航返回后协程仍在跑，极易泄漏。

- `viewModelScope` 需要 ViewModel 环境，**Composable 里拿不到**。

- `rememberCoroutineScope()` 返回的 Scope 跟随 **当前 Composable 所在的组合树**，UI 不在即取消，**最适合处理按钮点击、动画、短暂网络请求**。

##### 流式布局

- FlowRow

- FlowColumn

##### 自定义版式

##### 自适应布局

如果显示屏的最小宽度大于或等于 600dp，以 API 级别 36 为目标平台的应用可调整大小，并且能够进入多窗口模式（相当于 [`resizeableActivity="true"`](https://developer.android.google.cn/guide/topics/manifest/activity-element?hl=zh-cn#resizeableActivity)）。

避免尝试使用设备的实际屏幕尺寸。对于不同类型的屏幕，这会不准确；如果应用不是全屏显示，这也会不准确。

- 如果您想更改显示的*内容*，可以使用 [`BoxWithConstraints`](https://developer.android.google.cn/reference/kotlin/androidx/compose/foundation/layout/package-summary?hl=zh-cn#BoxWithConstraints(androidx.compose.ui.Modifier,androidx.compose.ui.Alignment,kotlin.Boolean,kotlin.Function1)) 作为更强大的替代方案。`BoxWithConstraints` 提供[测量约束条件](https://developer.android.google.cn/reference/kotlin/androidx/compose/foundation/layout/BoxWithConstraintsScope?hl=zh-cn)，您可以使用这些约束条件根据可用的显示空间调用不同的可组合项。但是，这样也会带来一些后果，因为 `BoxWithConstraints` 会将组合推迟到布局阶段（此时已知道这些约束条件），从而导致在布局期间执行更多工作。

**支持不同显示屏尺寸**

```kotlin
@Composable
fun Card(
    imageUrl: String,
    title: String,
    description: String
) {
    var showMore by remember { mutableStateOf(false) }

    BoxWithConstraints {
        if (maxWidth < 400.dp) {
            Column {
                Image(imageUrl)
                Title(title)
            }
        } else {
            Row {
                Column {
                    Title(title)
                    Description(
                        description = description,
                        showMore = showMore,
                        onShowMoreToggled = { newValue ->
                            showMore = newValue
                        }
                    )
                }
                Image(imageUrl)
            }
        }
    }
}
```

**使用窗口大小类别**

窗口大小类别将应用中可用的显示区域分类为：*紧凑*、*中等*、*展开*、*大*或*超大*。可用宽度和高度是单独分类的，因此在任何时间点，应用都有两个窗口大小类别：宽度窗口大小类别和高度窗口大小类别。但由于垂直滚动的普遍存在，可用宽度通常比可用高度更重要；因此，宽度窗口大小类别很可能与应用的界面更相关。

以下示例展示了如何计算窗口大小类别，并在窗口大小类别发生变化时随时接收更新：

```kotlin
val windowSizeClass = currentWindowAdaptiveInfo().windowSizeClass
```

**支持多窗口模式**

**构建列表详情布局**

```groovy
implementation("androidx.compose.material3.adaptive:adaptive")
implementation("androidx.compose.material3.adaptive:adaptive-layout")
implementation("androidx.compose.material3.adaptive:adaptive-navigation")
```

```kotlin
val scaffoldNavigator = rememberListDetailPaneScaffoldNavigator<MyItem>()
val scope = rememberCoroutineScope()
val backNavigationBehavior = BackNavigationBehavior.PopUntilScaffoldValueChange

NavigableListDetailPaneScaffold(
    navigator = scaffoldNavigator,
    listPane = {
        AnimatedPane {
            MyList(
                onItemClick = { item ->
                    // Navigate to the detail pane with the passed item
                    scope.launch {
                        scaffoldNavigator.navigateTo(
                            ListDetailPaneScaffoldRole.Detail,
                            item
                        )
                    }
                },
            )
        }
    },
    detailPane = {
        AnimatedPane {
            // Show the detail pane content if selected item is available
            scaffoldNavigator.currentDestination?.contentKey?.let {
                Column {
                    // Allow users to dismiss the detail pane. Use back navigation to
                    // hide an expanded detail pane.
                    if (scaffoldNavigator.scaffoldValue[ListDetailPaneScaffoldRole.Detail] == PaneAdaptedValue.Expanded) {
                        // Material design principles promote the usage of a right-aligned
                        // close (X) button.
                        IconButton(
                            modifier =  Modifier.align(Alignment.End).padding(16.dp),
                            onClick = {
                                scope.launch {
                                    scaffoldNavigator.navigateBack(backNavigationBehavior)
                                }
                            }
                        ) {
                            Icon(Icons.Default.Close, contentDescription = "Close")
                        }
                    }
                    MyDetails(it)
                }
            }
        }
    },
)
```

**让应用具备折叠感知能力**

**可见性监听**

```kotlin
var background by remember { mutableStateOf(PalePink) }
Card(
    modifier = modifier
        // ...
        .onVisibilityChanged(minDurationMs = 3000) {
            if (it) {
                background = MutedPlum
            }
        }
) {

    Box(
        modifier = Modifier
            // ...
            .background(background),
        contentAlignment = Alignment.Center,
    ) {
        // ...
    }
}
```

----

----

### 组件

##### 应用栏

- 顶部应用栏 `TopAppBar`

- 底部应用栏 `BootomAppBar`

单标题：

```kotlin
@Composable
fun SmallTopAppBarExample() {
    Scaffold(
        topBar = {
            TopAppBar(
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer,
                    titleContentColor = MaterialTheme.colorScheme.primary,
                ),
                title = {
                    Text("Small Top App Bar")
                }
            )
        },
    ) { innerPadding ->
        ScrollContent(innerPadding)
    }
}
```

居中标题：

```kotlin
@Composable
fun CenterAlignedTopAppBarExample() {
    val scrollBehavior = TopAppBarDefaults.pinnedScrollBehavior(rememberTopAppBarState())

    Scaffold(
        modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection),

        topBar = {
            CenterAlignedTopAppBar(
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer,
                    titleContentColor = MaterialTheme.colorScheme.primary,
                ),
                title = {
                    Text(
                        "Centered Top App Bar",
                        maxLines = 1,
                        overflow = TextOverflow.Ellipsis
                    )
                },
                navigationIcon = {
                    IconButton(onClick = { /* do something */ }) {
                        Icon(
                            imageVector = Icons.AutoMirrored.Filled.ArrowBack,
                            contentDescription = "Localized description"
                        )
                    }
                },
                actions = {
                    IconButton(onClick = { /* do something */ }) {
                        Icon(
                            imageVector = Icons.Filled.Menu,
                            contentDescription = "Localized description"
                        )
                    }
                },
                scrollBehavior = scrollBehavior,
            )
        },
    ) { innerPadding ->
        ScrollContent(innerPadding)
    }
}
```

##### 标志 BadgeBox

- **通知**：在应用图标或通知铃铛上显示未读通知的数量。
- **消息**：指示聊天应用中的新消息或未读消息。
- **状态更新**：显示任务的状态，例如“已完成”“正在进行中”或“失败”。
- **购物车数量**：显示用户购物车中的商品数量。
- **新内容**：突出显示可供用户使用的新内容或功能。

数量>5时显示红点：

```kotlin
@Composable
fun BadgeInteractiveExample() {
    var itemCount by remember { mutableIntStateOf(0) }

    Column(
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        BadgedBox(
            badge = {
                if (itemCount > 0) {
                    Badge(
                        containerColor = Color.Red,
                        contentColor = Color.White
                    ) {
                        Text("$itemCount")
                    }
                }
            }
        ) {
            Icon(
                imageVector = Icons.Filled.ShoppingCart,
                contentDescription = "Shopping cart",
            )
        }
        Button(onClick = { itemCount++ }) {
            Text("Add item")
        }
    }
}
```

##### Button

```kotlin
@Composable
fun FilledButtonExample(onClick: () -> Unit) {
    Button(onClick = { onClick() }) {
        Text("Filled")
    }
}


@Composable
fun FilledTonalButtonExample(onClick: () -> Unit) {
    FilledTonalButton(onClick = { onClick() }) {
        Text("Tonal")
    }
}

@Composable
fun OutlinedButtonExample(onClick: () -> Unit) {
    OutlinedButton(onClick = { onClick() }) {
        Text("Outlined")
    }
}

@Composable
fun ElevatedButtonExample(onClick: () -> Unit) {
    ElevatedButton(onClick = { onClick() }) {
        Text("Elevated")
    }
}


@Composable
fun TextButtonExample(onClick: () -> Unit) {
    TextButton(
        onClick = { onClick() }
    ) {
        Text("Text Button")
    }
}


@Composable
fun Example(onClick: () -> Unit) {
    FloatingActionButton(
        onClick = { onClick() },
    ) {
        Icon(Icons.Filled.Add, "Floating action button.")
    }
}


@Preview
@Composable
fun ToggleIconButtonExample() {
    // isToggled initial value should be read from a view model or persistent storage.
    var isToggled by rememberSaveable { mutableStateOf(false) }

    IconButton(
        onClick = { isToggled = !isToggled }
    ) {
        Icon(
            painter = if (isToggled) painterResource(R.drawable.favorite_filled) else painterResource(R.drawable.favorite),
            contentDescription = if (isToggled) "Selected icon button" else "Unselected icon button."
        )
    }
}
```

##### Card

- **`elevation`**：为组件添加阴影，使其看起来高于背景。
- **`colors`**：使用 `CardColors` 类型设置容器和所有子项的默认颜色。
- **`enabled`**：如果您为此参数传递 `false`，则卡片会显示为已停用，并且不会响应用户输入。
- **`onClick`**：通常情况下，`Card` 不接受点击事件。因此，您需要注意的主要重载是定义了 `onClick` 参数的重载。如果您希望 `Card` 的实现响应用户的按压操作，则应使用此重载。

```kotlin
//纯色卡片
@Composable
fun FilledCardExample() {
    Card(
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.surfaceVariant,
        ),
        modifier = Modifier
            .size(width = 240.dp, height = 100.dp)
    ) {
        Text(
            text = "Filled",
            modifier = Modifier
                .padding(16.dp),
            textAlign = TextAlign.Center,
        )
    }
}


//阴影卡片
@Composable
fun ElevatedCardExample() {
    ElevatedCard(
        elevation = CardDefaults.cardElevation(
            defaultElevation = 6.dp
        ),
        modifier = Modifier
            .size(width = 240.dp, height = 100.dp)
    ) {
        Text(
            text = "Elevated",
            modifier = Modifier
                .padding(16.dp),
            textAlign = TextAlign.Center,
        )
    }
}


//描边卡片
```

##### 轮播

```kotlin
@Composable
fun CarouselExample_MultiBrowse() {
    data class CarouselItem(
        val id: Int,
        @DrawableRes val imageResId: Int,
        val contentDescription: String
    )

    val items = remember {
        listOf(
            CarouselItem(0, R.drawable.cupcake, "cupcake"),
            CarouselItem(1, R.drawable.donut, "donut"),
            CarouselItem(2, R.drawable.eclair, "eclair"),
            CarouselItem(3, R.drawable.froyo, "froyo"),
            CarouselItem(4, R.drawable.gingerbread, "gingerbread"),
        )
    }

    HorizontalMultiBrowseCarousel(
        state = rememberCarouselState { items.count() },
        modifier = Modifier
            .fillMaxWidth()
            .wrapContentHeight()
            .padding(top = 16.dp, bottom = 16.dp),
        preferredItemWidth = 186.dp,
        itemSpacing = 8.dp,
        contentPadding = PaddingValues(horizontal = 16.dp)
    ) { i ->
        val item = items[i]
        Image(
            modifier = Modifier
                .height(205.dp)
                .maskClip(MaterialTheme.shapes.extraLarge),
            painter = painterResource(id = item.imageResId),
            contentDescription = item.contentDescription,
            contentScale = ContentScale.Crop
        )
    }
}
```

##### 复选框`CheckBox`

```kotlin
@Composable
fun CheckboxMinimalExample() {
    var checked by remember { mutableStateOf(true) }

    Row(
        verticalAlignment = Alignment.CenterVertically,
    ) {
        Text(
            "Minimal checkbox"
        )
        Checkbox(
            checked = checked,
            onCheckedChange = { checked = it }
        )
    }

    Text(
        if (checked) "Checkbox is checked" else "Checkbox is unchecked"
    )
}
```

##### Chip

类型：

- 辅助 `AssistChip`

- 过滤 `FilterChip`

- 输入 `InputChip`

- 建议 `SuggestionChip`

参数：

- **`label`**：显示在 chip 上的字符串。
- **`icon`**：显示在 chip 开头的图标。某些特定的可组合项具有单独的 `leadingIcon` 和 `trailingIcon` 参数。
- **`onClick`**：用户按下芯片时芯片调用的 lambda。

##### 日期选择器

##### 对话框

```kotlin
@Composable
fun AlertDialogExample(
    onDismissRequest: () -> Unit,
    onConfirmation: () -> Unit,
    dialogTitle: String,
    dialogText: String,
    icon: ImageVector,
) {
    AlertDialog(
        icon = {
            Icon(icon, contentDescription = "Example Icon")
        },
        title = {
            Text(text = dialogTitle)
        },
        text = {
            Text(text = dialogText)
        },
        onDismissRequest = {
            onDismissRequest()
        },
        confirmButton = {
            TextButton(
                onClick = {
                    onConfirmation()
                }
            ) {
                Text("Confirm")
            }
        },
        dismissButton = {
            TextButton(
                onClick = {
                    onDismissRequest()
                }
            ) {
                Text("Dismiss")
            }
        }
    )
}
```

##### 分隔线

```kotlin
//水平分隔线
@Composable
fun HorizontalDividerExample() {
    Column(
        verticalArrangement = Arrangement.spacedBy(8.dp),
    ) {
        Text("First item in list")
        HorizontalDivider(thickness = 2.dp)
        Text("Second item in list")
    }
}


//垂直分隔线
@Composable
fun VerticalDividerExample() {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .height(IntrinsicSize.Min),
        horizontalArrangement = Arrangement.SpaceEvenly
    ) {
        Text("First item in row")
        VerticalDivider(color = MaterialTheme.colorScheme.secondary)
        Text("Second item in row")
    }
}
```

##### 列表和网格

列表使用`Column`和`Row`，用`verticalScroll()`修饰符使`Column`可滚动。

[`LazyColumn`](https://developer.android.google.cn/reference/kotlin/androidx/compose/foundation/lazy/package-summary?hl=zh-cn#LazyColumn(androidx.compose.ui.Modifier,androidx.compose.foundation.lazy.LazyListState,androidx.compose.foundation.layout.PaddingValues,kotlin.Boolean,androidx.compose.foundation.layout.Arrangement.Vertical,androidx.compose.ui.Alignment.Horizontal,androidx.compose.foundation.gestures.FlingBehavior,kotlin.Boolean,kotlin.Function1)) 和 [`LazyRow`](https://developer.android.google.cn/reference/kotlin/androidx/compose/foundation/lazy/package-summary?hl=zh-cn#LazyRow(androidx.compose.ui.Modifier,androidx.compose.foundation.lazy.LazyListState,androidx.compose.foundation.layout.PaddingValues,kotlin.Boolean,androidx.compose.foundation.layout.Arrangement.Horizontal,androidx.compose.ui.Alignment.Vertical,androidx.compose.foundation.gestures.FlingBehavior,kotlin.Boolean,kotlin.Function1)) 之间的区别就在于它们的列表项布局和滚动方向不同。[`LazyColumn`](https://developer.android.google.cn/reference/kotlin/androidx/compose/foundation/lazy/package-summary?hl=zh-cn#LazyColumn(androidx.compose.ui.Modifier,androidx.compose.foundation.lazy.LazyListState,androidx.compose.foundation.layout.PaddingValues,kotlin.Boolean,androidx.compose.foundation.layout.Arrangement.Vertical,androidx.compose.ui.Alignment.Horizontal,androidx.compose.foundation.gestures.FlingBehavior,kotlin.Boolean,kotlin.Function1)) 生成的是垂直滚动列表，而 [`LazyRow`](https://developer.android.google.cn/reference/kotlin/androidx/compose/foundation/lazy/package-summary?hl=zh-cn#LazyRow(androidx.compose.ui.Modifier,androidx.compose.foundation.lazy.LazyListState,androidx.compose.foundation.layout.PaddingValues,kotlin.Boolean,androidx.compose.foundation.layout.Arrangement.Horizontal,androidx.compose.ui.Alignment.Vertical,androidx.compose.foundation.gestures.FlingBehavior,kotlin.Boolean,kotlin.Function1)) 生成的是水平滚动列表。

```kotlin
LazyColumn {
    // Add a single item
    item {
        Text(text = "First item")
    }

    // Add 5 items
    items(5) { index ->
        Text(text = "Item: $index")
    }

    // Add another single item
    item {
        Text(text = "Last item")
    }
}

LazyVerticalGrid(
    columns = GridCells.Adaptive(minSize = 128.dp)
) {
    items(photos) { photo ->
        PhotoItem(photo)
    }
}

LazyVerticalGrid(
    columns = GridCells.Adaptive(minSize = 30.dp)
) {
    item(span = {
        // LazyGridItemSpanScope:
        // maxLineSpan
        GridItemSpan(maxLineSpan)
    }) {
        CategoryCard("Fruits")
    }
    // ...
}

LazyVerticalStaggeredGrid(
    columns = StaggeredGridCells.Adaptive(200.dp),
    verticalItemSpacing = 4.dp,
    horizontalArrangement = Arrangement.spacedBy(4.dp),
    content = {
        items(randomSizedPhotos) { photo ->
            AsyncImage(
                model = photo,
                contentScale = ContentScale.Crop,
                contentDescription = null,
                modifier = Modifier
                    .fillMaxWidth()
                    .wrapContentHeight()
            )
        }
    },
    modifier = Modifier.fillMaxSize()
)


//内容内边距
LazyColumn(
    contentPadding = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
) {
    // ...
}


//内容间距
LazyColumn(
    verticalArrangement = Arrangement.spacedBy(4.dp),
) {
    // ...
}
```

##### 导航栏

使用 [`NavigationBar`](https://developer.android.google.cn/reference/kotlin/androidx/compose/material3/package-summary?hl=zh-cn#NavigationBar(androidx.compose.ui.Modifier,androidx.compose.ui.graphics.Color,androidx.compose.ui.graphics.Color,androidx.compose.ui.unit.Dp,androidx.compose.foundation.layout.WindowInsets,kotlin.Function1)) 和 [`NavigationBarItem`](https://developer.android.google.cn/reference/kotlin/androidx/compose/material3/package-summary?hl=zh-cn#(androidx.compose.foundation.layout.RowScope).NavigationBarItem(kotlin.Boolean,kotlin.Function0,kotlin.Function0,androidx.compose.ui.Modifier,kotlin.Boolean,kotlin.Function0,kotlin.Boolean,androidx.compose.material3.NavigationBarItemColors,androidx.compose.foundation.interaction.MutableInteractionSource)) 可组合项来实现目的地切换逻辑。每个 `NavigationBarItem` 都代表一个目的地。

`NavigationBarItem` 包含以下关键参数：

- `selected`：确定当前项是否以醒目方式突出显示。
- `onClick()`：定义当用户点击相应项时要执行的操作。用于处理导航事件、更新所选项目状态或加载相应内容的逻辑应放在此处。
- `label`：在商品内显示文字。可选。
- `icon`：在商品内显示图标。可选。

```kotlin
@Composable
fun NavigationBarExample(modifier: Modifier = Modifier) {
    val navController = rememberNavController()
    val startDestination = Destination.SONGS
    var selectedDestination by rememberSaveable { mutableIntStateOf(startDestination.ordinal) }

    Scaffold(
        modifier = modifier,
        bottomBar = {
            NavigationBar(windowInsets = NavigationBarDefaults.windowInsets) {
                Destination.entries.forEachIndexed { index, destination ->
                    NavigationBarItem(
                        selected = selectedDestination == index,
                        onClick = {
                            navController.navigate(route = destination.route)
                            selectedDestination = index
                        },
                        icon = {
                            Icon(
                                destination.icon,
                                contentDescription = destination.contentDescription
                            )
                        },
                        label = { Text(destination.label) }
                    )
                }
            }
        }
    ) { contentPadding ->
        AppNavHost(navController, startDestination, modifier = Modifier.padding(contentPadding))
    }
}
```

##### 进度条

- 确定性指示器

- 不确定性指示器

```kotlin
@Composable
fun LinearDeterminateIndicator() {
    var currentProgress by remember { mutableFloatStateOf(0f) }
    var loading by remember { mutableStateOf(false) }
    val scope = rememberCoroutineScope() // Create a coroutine scope

    Column(
        verticalArrangement = Arrangement.spacedBy(12.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        modifier = Modifier.fillMaxWidth()
    ) {
        Button(onClick = {
            loading = true
            scope.launch {
                loadProgress { progress ->
                    currentProgress = progress
                }
                loading = false // Reset loading when the coroutine finishes
            }
        }, enabled = !loading) {
            Text("Start loading")
        }

        if (loading) {
            LinearProgressIndicator(
                progress = { currentProgress },
                modifier = Modifier.fillMaxWidth(),
            )
        }
    }
}

/** Iterate the progress value */
suspend fun loadProgress(updateProgress: (Float) -> Unit) {
    for (i in 1..100) {
        updateProgress(i.toFloat() / 100)
        delay(100)
    }
}
```

```kotlin
@Composable
fun IndeterminateCircularIndicator() {
    var loading by remember { mutableStateOf(false) }

    Button(onClick = { loading = true }, enabled = !loading) {
        Text("Start loading")
    }

    if (!loading) return

    CircularProgressIndicator(
        modifier = Modifier.width(64.dp),
        color = MaterialTheme.colorScheme.secondary,
        trackColor = MaterialTheme.colorScheme.surfaceVariant,
    )
}
```

##### 下拉刷新

使用 [`PullToRefreshBox`](https://developer.android.google.cn/reference/kotlin/androidx/compose/material3/pulltorefresh/package-summary?hl=zh-cn#PullToRefreshBox(kotlin.Boolean,kotlin.Function0,androidx.compose.ui.Modifier,androidx.compose.material3.pulltorefresh.PullToRefreshState,androidx.compose.ui.Alignment,kotlin.Function1,kotlin.Function1)) 可组合项来实现下拉刷新，该可组合项充当可滚动内容的容器。以下关键参数可控制刷新行为和外观：

- `isRefreshing`：一个布尔值，用于指示刷新操作是否正在进行。
- `onRefresh`：当用户发起刷新时执行的 lambda 函数。
- `indicator`：自定义系统在下拉刷新时绘制的指示器。

```kotlin
@Composable
fun PullToRefreshBasicSample(
    items: List<String>,
    isRefreshing: Boolean,
    onRefresh: () -> Unit,
    modifier: Modifier = Modifier
) {
    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = onRefresh,
        modifier = modifier
    ) {
        LazyColumn(Modifier.fillMaxSize()) {
            items(items) {
                ListItem({ Text(text = it) })
            }
        }
    }
}
```

##### 单选按钮

RadioButton

##### 切换按钮Switch

---------

-----

### 主题

----

-----

### 文本和排版

```kotlin
@Composable
fun StringResourceText() {
    Text(
        stringResource(R.string.hello_world),
        color = Color.Blue,
        fontSize = 30.sp,
        fontStyle = FontStyle.Italic,
        fontWeight = FontWeight.Bold    
    )
}


//添加多种样式
@Composable
fun MultipleStylesInText() {
    Text(
        buildAnnotatedString {
            withStyle(style = SpanStyle(color = Color.Blue)) {
                append("H")
            }
            append("ello ")

            withStyle(style = SpanStyle(fontWeight = FontWeight.Bold, color = Color.Red)) {
                append("W")
            }
            append("orld")
        }
    )
}


//带链接
@Composable
fun AnnotatedHtmlStringWithLink(
    modifier: Modifier = Modifier,
    htmlText: String = """
       <h1>Jetpack Compose</h1>
       <p>
           Build <b>better apps</b> faster with <a href="https://www.android.com">Jetpack Compose</a>
       </p>
    """.trimIndent()
) {
    Text(
        AnnotatedString.fromHtml(
            htmlText,
            linkStyles = TextLinkStyles(
                style = SpanStyle(
                    textDecoration = TextDecoration.Underline,
                    fontStyle = FontStyle.Italic,
                    color = Color.Blue
                )
            )
        ),
        modifier
    )
}
```

---

------

### 图片和图形

加载图片

```kotlin
Image(
    painter = painterResource(id = R.drawable.dog),
    contentDescription = stringResource(id = R.string.dog_content_description)
)
```

### 可绘制对象支持

`painterResource` 目前支持以下可绘制对象类型：

- [`AnimatedVectorDrawable`](https://developer.android.google.cn/reference/android/graphics/drawable/AnimatedVectorDrawable?hl=zh-cn)
- [`BitmapDrawable`](https://developer.android.google.cn/reference/android/graphics/drawable/BitmapDrawable?hl=zh-cn)（PNG、JPG、WEBP）
- [`ColorDrawable`](https://developer.android.google.cn/reference/android/graphics/drawable/ColorDrawable?hl=zh-cn)
- [`VectorDrawable`](https://developer.android.google.cn/reference/android/graphics/drawable/VectorDrawable?hl=zh-cn)

##### 从互联网加载图片

```kotlin
AsyncImage(
    model = "https://example.com/image.jpg",
    contentDescription = "Translated description of what the image contains"
)
```

缩放：

```kotlin
val imageModifier = Modifier
    .size(150.dp)
    .border(BorderStroke(1.dp, Color.Black))
    .background(Color.Yellow)
Image(
    painter = painterResource(id = R.drawable.dog),
    contentDescription = stringResource(id = R.string.dog_content_description),
    contentScale = ContentScale.Fit,
    modifier = imageModifier
)
```

裁剪：

```kotlin
Image(
    painter = painterResource(id = R.drawable.dog),
    contentDescription = stringResource(id = R.string.dog_content_description),
    contentScale = ContentScale.Crop,
    modifier = Modifier
        .size(200.dp)
        .border(        //边框
            BorderStroke(borderWidth, Color.Yellow),
            CircleShape
        )
        .clip(CircleShape)
)

//指定圆角
Modifier.clip(RoundedCornerShape(16.dp)

//自定义宽高比
Modifier.aspectRatio(16f / 9f)

//模糊处理
Mofifier.blur(
            radiusX = 10.dp,
            radiusY = 10.dp,
            edgeTreatment = BlurredEdgeTreatment(RoundedCornerShape(8.dp))
        )
```

#### 优化图片性能

- 尽可能缩小图片文件

- 使用WEBP格式，而不是JPEG或PNG

- 针对屏幕分辨率适配图片

- 使用图片加载库，按比例缩放图片以适应屏幕上的视图大小。

#### 图形修饰符

- [`drawWithContent`](https://developer.android.google.cn/reference/kotlin/androidx/compose/ui/draw/package-summary?hl=zh-cn#(androidx.compose.ui.Modifier).drawWithContent(kotlin.Function1)) ： 选择绘制顺序

- [`drawBehind`](https://developer.android.google.cn/reference/kotlin/androidx/compose/ui/draw/package-summary?hl=zh-cn#(androidx.compose.ui.Modifier).drawBehind(kotlin.Function1)) ：在可组合项后面绘制

- [`drawWithCache`](https://developer.android.google.cn/reference/kotlin/androidx/compose/ui/draw/package-summary?hl=zh-cn#(androidx.compose.ui.Modifier).drawWithCache(kotlin.Function1)) ：绘制和缓存绘制对象

- [`Modifier.graphicsLayer`](https://developer.android.google.cn/develop/ui/compose/graphics/draw/modifiers?hl=zh-cn#graphicsLayer)：对可组合项应用转换
  
  - `scaleX`、`scaleY`：缩放
  
  - `translationX`、`translationY`：平移
  
  - `rotationX`、`rotationY`、`rotationZ`：旋转
  
  - `transformOrigin`：原点，默认为(0.5f,0.5f)
  
  - `clip`：裁剪，区别于`Modifier.clip`
  
  - `alpha`：不透明度

#### 画笔`Brush`

```kotlin
val brush = Brush.horizontalGradient(listOf(Color.Red, Color.Blue))
Canvas(
    modifier = Modifier.size(200.dp),
    onDraw = {
        drawCircle(brush)
    }
)
```

**更改笔刷大小：**

```kotlin
val listColors = listOf(Color.Yellow, Color.Red, Color.Blue)
val customBrush = remember {
    object : ShaderBrush() {
        override fun createShader(size: Size): Shader {
            return LinearGradientShader(
                colors = listColors,
                from = Offset.Zero,
                to = Offset(size.width / 4f, 0f),
                tileMode = TileMode.Mirror
            )
        }
    }
}
Box(
    modifier = Modifier
        .requiredSize(200.dp)
        .background(customBrush)
)
```

**使用图片作为笔刷：**

```kotlin
val imageBrush =
    ShaderBrush(ImageShader(ImageBitmap.imageResource(id = R.drawable.dog)))

// Use ImageShader Brush with background
Box(
    modifier = Modifier
        .requiredSize(200.dp)
        .background(imageBrush)
)

// Use ImageShader Brush with TextStyle
Text(
    text = "Hello Android!",
    style = TextStyle(
        brush = imageBrush,
        fontWeight = FontWeight.ExtraBold,
        fontSize = 36.sp
    )
)

// Use ImageShader Brush with DrawScope#drawCircle()
Canvas(onDraw = {
    drawCircle(imageBrush)
}, modifier = Modifier.size(200.dp))
```

#### 形状

#### 影子

- `Modifier.shadow()`

- `Modifier.dropShadow()`

- `Moifier.innerShadow()`：内阴影

```kotlin
@Composable
fun ElevationBasedShadow() {
    Box(
        modifier = Modifier.aspectRatio(1f).fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        Box(
            Modifier
                .size(100.dp, 100.dp)
                .shadow(10.dp, RectangleShape)
                .background(Color.White)
        )
    }
}
```

使用 [`dropShadow()`](https://developer.android.google.cn/reference/kotlin/androidx/compose/ui/Modifier?hl=zh-cn#(androidx.compose.ui.Modifier).dropShadow(androidx.compose.ui.graphics.Shape,kotlin.Function1)) 修饰符可在内容后面绘制精确的阴影，使元素看起来具有立体感。

您可以通过其 `Shadow` 参数控制以下关键方面：

- `radius`：定义模糊效果的柔和度和扩散度。
- `color`：定义色调的颜色。
- `offset`：沿 x 轴和 y 轴定位阴影的几何图形。
- `spread`：控制阴影几何图形的扩展或收缩。

```kotlin
@Composable
fun SimpleDropShadowUsage() {
    Box(Modifier.fillMaxSize()) {
        Box(
            Modifier
                .width(300.dp)
                .height(300.dp)
                .dropShadow(
                    shape = RoundedCornerShape(20.dp),
                    shadow = Shadow(
                        radius = 10.dp,
                        spread = 6.dp,
                        color = Color(0x40000000),
                        offset = DpOffset(x = 4.dp, 4.dp)
                    )
                )
                .align(Alignment.Center)
                .background(
                    color = Color.White,
                    shape = RoundedCornerShape(20.dp)
                )
        ) {
            Text(
                "Drop Shadow",
                modifier = Modifier.align(Alignment.Center),
                fontSize = 32.sp
            )
        }
    }
}
```

### 代码要点

- `dropShadow()` 修饰符应用于内部 `Box`。影子具有以下特征：
  - 圆角矩形形状 (`RoundedCornerShape(20.dp)`)
  - 模糊半径为 `10.dp`，使边缘柔和而模糊
  - 扩散半径为 `6.dp`，可扩大阴影的大小，使其大于投射阴影的盒子
  - alpha 为 `0.5f`，使阴影半透明
- 定义阴影后，.应用了 `background()` 修饰符。
  - `Box`填充了白色。
  - 背景被剪裁为与阴影相同的圆角矩形形状。

内阴影：

```kotlin
@Composable
fun SimpleInnerShadowUsage() {
    Box(Modifier.fillMaxSize()) {
        Box(
            Modifier
                .width(300.dp)
                .height(200.dp)
                .align(Alignment.Center)
                // note that the background needs to be defined before defining the inner shadow
                .background(
                    color = Color.White,
                    shape = RoundedCornerShape(20.dp)
                )
                .innerShadow(
                    shape = RoundedCornerShape(20.dp),
                    shadow = Shadow(
                        radius = 10.dp,
                        spread = 2.dp,
                        color = Color(0x40000000),
                        offset = DpOffset(x = 6.dp, 7.dp)
                    )
                )

        ) {
            Text(
                "Inner Shadow",
                modifier = Modifier.align(Alignment.Center),
                fontSize = 32.sp
            )
        }
    }
}
```

------

-------

### 动画

-----

-------

### 无障碍

-----

-------

### 轻触和输入

------

---------

### 性能

------

------

### 样式指南





---

---

## 页面状态

为了跟踪页面状态变化，我们必须使用 remember 和 mutableStateOf 函数。

Composable 函数可以通过使用 remember 将本地状态存储在内存中，并跟踪传递给 mutableStateOf 的值的变化。当值被更新时，使用该状态的 Composable 函数（及其子函数）将被自动重新绘制。我们把这称为重组（recomposition）。
