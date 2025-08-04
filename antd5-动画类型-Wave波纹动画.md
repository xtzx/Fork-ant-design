# Ant Design 5 Wave 波纹动画深度分析

## 目录
- [动画概览](#动画概览)
- [实现原理](#实现原理)
- [技术架构](#技术架构)
- [核心组件分析](#核心组件分析)
- [使用场景分析](#使用场景分析)
- [性能优化](#性能优化)
- [实际应用案例](#实际应用案例)
- [个人设计思考](#个人设计思考)
- [优化建议](#优化建议)
- [潜在问题](#潜在问题)
- [学习价值](#学习价值)

## 动画概览

Wave 波纹动画是 Ant Design 5 中最具交互性和视觉反馈的动画效果，主要用于按钮点击、选择框勾选等用户交互场景，通过模拟水波纹扩散效果提供直观的操作反馈。

### 核心特点
- **交互触发**：通过用户点击等交互动作触发
- **物理模拟**：模拟真实的水波纹扩散效果
- **动态计算**：根据元素大小和形状动态生成波纹
- **视觉反馈**：为用户操作提供即时的视觉确认

### 视觉效果特征
- **从点到圆**：从点击位置开始向外扩散
- **渐变透明**：扩散过程中透明度逐渐降低
- **尺寸自适应**：根据目标元素大小调整波纹范围
- **形状匹配**：适配元素的边框圆角等形状特征

## 实现原理

### 1. 核心实现文件分析

```typescript
// components/_util/wave/WaveEffect.tsx
const WaveEffect = (props: WaveEffectProps) => {
  const { className, target, component, registerUnmount } = props;
  const divRef = React.useRef<HTMLDivElement>(null);

  // 波纹状态管理
  const [color, setWaveColor] = React.useState<string | null>(null);
  const [borderRadius, setBorderRadius] = React.useState<number[]>([]);
  const [left, setLeft] = React.useState(0);
  const [top, setTop] = React.useState(0);
  const [width, setWidth] = React.useState(0);
  const [height, setHeight] = React.useState(0);
  const [enabled, setEnabled] = React.useState(false);

  // 动态样式计算
  const waveStyle: React.CSSProperties = {
    left,
    top,
    width,
    height,
    borderRadius: borderRadius.map((radius) => `${radius}px`).join(' '),
  };

  if (color) {
    waveStyle['--wave-color'] = color; // CSS 变量传递颜色
  }

  // 同步位置和样式
  function syncPos() {
    const nodeStyle = getComputedStyle(target);

    // 获取波纹颜色
    setWaveColor(getTargetWaveColor(target));

    const isStatic = nodeStyle.position === 'static';
    const { borderLeftWidth, borderTopWidth } = nodeStyle;

    // 计算波纹位置
    setLeft(isStatic ? target.offsetLeft : validateNum(-parseFloat(borderLeftWidth)));
    setTop(isStatic ? target.offsetTop : validateNum(-parseFloat(borderTopWidth)));
    setWidth(target.offsetWidth);
    setHeight(target.offsetHeight);

    // 获取边框圆角并应用到波纹
    const {
      borderTopLeftRadius,
      borderTopRightRadius,
      borderBottomLeftRadius,
      borderBottomRightRadius,
    } = nodeStyle;

    setBorderRadius([
      borderTopLeftRadius,
      borderTopRightRadius,
      borderBottomRightRadius,
      borderBottomLeftRadius,
    ].map((radius) => validateNum(parseFloat(radius))));
  }

  return (
    <CSSMotion
      visible
      motionAppear
      motionName="wave-motion"
      motionDeadline={5000}
      onAppearEnd={(_, event) => {
        if (event.deadline || (event as TransitionEvent).propertyName === 'opacity') {
          const holder = divRef.current?.parentElement!;
          unmountRef.current?.().then(() => {
            holder?.remove(); // 动画结束后清理 DOM
          });
        }
        return false;
      }}
    >
      {({ className: motionClassName }, ref) => (
        <div
          ref={composeRef(divRef, ref)}
          className={classNames(className, motionClassName, {
            'wave-quick': isSmallComponent // 小组件使用快速动画
          })}
          style={waveStyle}
        />
      )}
    </CSSMotion>
  );
};
```

### 2. 波纹触发机制

```typescript
// components/_util/wave/WaveEffect.tsx
const showWaveEffect: ShowWaveEffect = (target, info) => {
  const { component } = info;

  // 特殊处理：跳过未勾选的复选框
  if (component === 'Checkbox' && !target.querySelector<HTMLInputElement>('input')?.checked) {
    return;
  }

  // 创建波纹容器
  const holder = document.createElement('div');
  holder.style.position = 'absolute';
  holder.style.left = '0px';
  holder.style.top = '0px';
  target?.insertBefore(holder, target?.firstChild);

  const reactRender = unstableSetRender();
  let unmountCallback: UnmountType | null = null;

  function registerUnmount() {
    return unmountCallback;
  }

  // 渲染波纹效果
  unmountCallback = reactRender(
    <WaveEffect {...info} target={target} registerUnmount={registerUnmount} />,
    holder,
  );
};
```

### 3. 颜色计算逻辑

```typescript
// components/_util/wave/util.ts
export const getTargetWaveColor = (node: HTMLElement): string | null => {
  const { borderTopColor, borderColor } = getComputedStyle(node);

  // 优先使用边框颜色
  if (borderTopColor && borderTopColor !== 'rgba(0, 0, 0, 0)') {
    return borderTopColor;
  }

  if (borderColor && borderColor !== 'rgba(0, 0, 0, 0)') {
    return borderColor;
  }

  // fallback 到默认主题色
  return null;
};
```

### 4. CSS 波纹动画定义

```css
/* 实际生成的波纹动画样式 */
.wave-motion-appear {
  animation-name: waveEffect;
  animation-duration: 0.4s;
  animation-timing-function: cubic-bezier(0.08, 0.82, 0.17, 1);
  animation-fill-mode: forwards;
}

.wave-quick.wave-motion-appear {
  animation-duration: 0.2s;
}

@keyframes waveEffect {
  0% {
    transform: scale(0);
    opacity: 0.5;
  }
  100% {
    transform: scale(1);
    opacity: 0;
  }
}

/* 波纹层样式 */
.ant-wave {
  position: absolute;
  background: radial-gradient(circle, var(--wave-color, #1677ff) 10%, transparent 10.01%);
  transform: scale(0);
  pointer-events: none;
  box-sizing: border-box;
}
```

## 技术架构

### 1. 架构层次图

```
┌─────────────────────────────────────────────────────────┐
│                     触发层                              │
│        Button Click │ Checkbox Check │ Input Focus      │
├─────────────────────────────────────────────────────────┤
│                     管理层                              │
│              showWaveEffect Function                    │
├─────────────────────────────────────────────────────────┤
│                     计算层                              │
│    Position Calc │ Size Calc │ Color Calc │ Shape Calc │
├─────────────────────────────────────────────────────────┤
│                     渲染层                              │
│             WaveEffect Component                        │
├─────────────────────────────────────────────────────────┤
│                     动画层                              │
│          CSSMotion + CSS Animations                     │
├─────────────────────────────────────────────────────────┤
│                     清理层                              │
│               DOM Cleanup + Memory GC                   │
└─────────────────────────────────────────────────────────┘
```

### 2. 生命周期流程

```typescript
// Wave 动画的完整生命周期
interface WaveLifeCycle {
  // 1. 触发阶段
  trigger: {
    event: MouseEvent | TouchEvent;
    target: HTMLElement;
    component: string;
  };

  // 2. 初始化阶段
  initialize: {
    createContainer: () => HTMLElement;
    calculatePosition: () => { left: number; top: number; width: number; height: number };
    extractStyle: () => { color: string; borderRadius: number[] };
  };

  // 3. 渲染阶段
  render: {
    createWaveElement: () => React.ReactElement;
    applyStyles: (styles: WaveStyles) => void;
    startAnimation: () => void;
  };

  // 4. 动画阶段
  animate: {
    expand: () => void;    // 波纹扩散
    fadeOut: () => void;   // 透明度渐变
  };

  // 5. 清理阶段
  cleanup: {
    removeElement: () => void;
    clearMemory: () => void;
    unregisterCallbacks: () => void;
  };
}
```

### 3. 状态管理模式

```typescript
// 波纹状态管理
interface WaveState {
  // 几何状态
  geometry: {
    left: number;
    top: number;
    width: number;
    height: number;
    borderRadius: number[];
  };

  // 视觉状态
  appearance: {
    color: string | null;
    opacity: number;
    scale: number;
  };

  // 控制状态
  control: {
    enabled: boolean;
    animating: boolean;
    destroyed: boolean;
  };
}

// 状态更新器
const useWaveState = (target: HTMLElement) => {
  const [state, setState] = useState<WaveState>(initialState);

  const updateGeometry = useCallback(() => {
    setState(prev => ({
      ...prev,
      geometry: calculateGeometry(target),
    }));
  }, [target]);

  const updateAppearance = useCallback((appearance: Partial<WaveState['appearance']>) => {
    setState(prev => ({
      ...prev,
      appearance: { ...prev.appearance, ...appearance },
    }));
  }, []);

  return { state, updateGeometry, updateAppearance };
};
```

## 核心组件分析

### 1. WaveEffect 组件深度解析

```typescript
// WaveEffect 组件的核心功能分解
const WaveEffect = (props: WaveEffectProps) => {
  // === 状态管理 ===
  const [waveState, setWaveState] = useState<WaveState>({
    geometry: { left: 0, top: 0, width: 0, height: 0, borderRadius: [] },
    appearance: { color: null, opacity: 0.5, scale: 0 },
    control: { enabled: false, animating: false, destroyed: false }
  });

  // === 位置同步逻辑 ===
  const syncPosition = useCallback(() => {
    const computedStyle = getComputedStyle(props.target);
    const isStatic = computedStyle.position === 'static';

    // 1. 计算波纹容器位置
    const position = {
      left: isStatic ? props.target.offsetLeft : -parseFloat(computedStyle.borderLeftWidth),
      top: isStatic ? props.target.offsetTop : -parseFloat(computedStyle.borderTopWidth),
      width: props.target.offsetWidth,
      height: props.target.offsetHeight,
    };

    // 2. 提取边框圆角
    const borderRadius = [
      'borderTopLeftRadius',
      'borderTopRightRadius',
      'borderBottomRightRadius',
      'borderBottomLeftRadius'
    ].map(prop => parseFloat(computedStyle[prop as any]) || 0);

    // 3. 获取波纹颜色
    const color = getTargetWaveColor(props.target);

    setWaveState(prev => ({
      ...prev,
      geometry: { ...position, borderRadius },
      appearance: { ...prev.appearance, color }
    }));
  }, [props.target]);

  // === 响应式监听 ===
  useEffect(() => {
    // 延迟同步，等待 DOM 更新
    const syncId = raf(() => {
      syncPosition();
      setWaveState(prev => ({ ...prev, control: { ...prev.control, enabled: true } }));
    });

    // 监听尺寸变化
    let resizeObserver: ResizeObserver | undefined;
    if (typeof ResizeObserver !== 'undefined') {
      resizeObserver = new ResizeObserver(syncPosition);
      resizeObserver.observe(props.target);
    }

    return () => {
      raf.cancel(syncId);
      resizeObserver?.disconnect();
    };
  }, [syncPosition]);

  // === 渲染优化 ===
  if (!waveState.control.enabled) {
    return null; // 避免不必要的渲染
  }

  const isSmallComponent = (props.component === 'Checkbox' || props.component === 'Radio') &&
                          props.target?.classList.contains(TARGET_CLS);

  return (
    <CSSMotion
      visible
      motionAppear
      motionName="wave-motion"
      motionDeadline={5000}
      onAppearEnd={handleAnimationEnd}
    >
      {({ className, style }, ref) => (
        <div
          ref={composeRef(divRef, ref)}
          className={classNames(props.className, className, {
            'wave-quick': isSmallComponent
          })}
          style={{
            ...waveState.geometry,
            borderRadius: waveState.geometry.borderRadius.map(r => `${r}px`).join(' '),
            '--wave-color': waveState.appearance.color,
            ...style,
          }}
        />
      )}
    </CSSMotion>
  );
};
```

### 2. 波纹管理器设计

```typescript
// 全局波纹管理器
class WaveManager {
  private static instance: WaveManager;
  private activeWaves = new Map<HTMLElement, WaveInstance>();
  private config: WaveConfig = {
    maxConcurrent: 10,
    defaultDuration: 400,
    quickDuration: 200,
    cleanupDelay: 100,
  };

  static getInstance(): WaveManager {
    if (!WaveManager.instance) {
      WaveManager.instance = new WaveManager();
    }
    return WaveManager.instance;
  }

  // 创建波纹效果
  createWave(target: HTMLElement, options: WaveOptions = {}): WaveInstance {
    // 清理已存在的波纹
    this.cleanupWave(target);

    // 检查并发限制
    if (this.activeWaves.size >= this.config.maxConcurrent) {
      this.cleanupOldestWave();
    }

    const waveInstance = new WaveInstance(target, {
      ...this.config,
      ...options,
      onComplete: () => {
        this.activeWaves.delete(target);
        options.onComplete?.();
      }
    });

    this.activeWaves.set(target, waveInstance);
    waveInstance.start();

    return waveInstance;
  }

  // 清理指定元素的波纹
  cleanupWave(target: HTMLElement): void {
    const existingWave = this.activeWaves.get(target);
    if (existingWave) {
      existingWave.destroy();
      this.activeWaves.delete(target);
    }
  }

  // 清理最旧的波纹
  private cleanupOldestWave(): void {
    const firstEntry = this.activeWaves.entries().next();
    if (!firstEntry.done) {
      const [target, wave] = firstEntry.value;
      wave.destroy();
      this.activeWaves.delete(target);
    }
  }

  // 清理所有波纹
  cleanupAll(): void {
    for (const [target, wave] of this.activeWaves) {
      wave.destroy();
    }
    this.activeWaves.clear();
  }
}

// 波纹实例
class WaveInstance {
  private element: HTMLElement;
  private container: HTMLElement | null = null;
  private animation: Animation | null = null;
  private config: WaveConfig;
  private destroyed = false;

  constructor(element: HTMLElement, config: WaveConfig) {
    this.element = element;
    this.config = config;
  }

  start(): void {
    if (this.destroyed) return;

    this.createContainer();
    this.calculateStyles();
    this.startAnimation();
  }

  destroy(): void {
    if (this.destroyed) return;

    this.destroyed = true;
    this.animation?.cancel();
    this.container?.remove();
    this.config.onComplete?.();
  }

  private createContainer(): void {
    this.container = document.createElement('div');
    this.container.className = 'ant-wave';
    this.container.style.cssText = `
      position: absolute;
      left: 0;
      top: 0;
      pointer-events: none;
    `;

    this.element.insertBefore(this.container, this.element.firstChild);
  }

  private calculateStyles(): void {
    if (!this.container) return;

    const computedStyle = getComputedStyle(this.element);
    const rect = this.element.getBoundingClientRect();

    // 应用计算出的样式
    Object.assign(this.container.style, {
      width: `${rect.width}px`,
      height: `${rect.height}px`,
      borderRadius: computedStyle.borderRadius,
      background: `radial-gradient(circle, ${this.getWaveColor()} 10%, transparent 10.01%)`,
    });
  }

  private startAnimation(): void {
    if (!this.container || this.destroyed) return;

    this.animation = this.container.animate([
      { transform: 'scale(0)', opacity: '0.5' },
      { transform: 'scale(1)', opacity: '0' }
    ], {
      duration: this.config.duration,
      easing: 'cubic-bezier(0.08, 0.82, 0.17, 1)',
      fill: 'forwards'
    });

    this.animation.finished.then(() => {
      if (!this.destroyed) {
        this.destroy();
      }
    });
  }

  private getWaveColor(): string {
    const computedStyle = getComputedStyle(this.element);
    return computedStyle.borderTopColor || computedStyle.borderColor || '#1677ff';
  }
}
```

## 使用场景分析

### 1. Button 按钮点击波纹

```typescript
// components/button/button.tsx 中的波纹集成
const Button = React.forwardRef<HTMLButtonElement, ButtonProps>((props, ref) => {
  const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
    // 触发波纹效果
    if (buttonRef.current) {
      showWaveEffect(buttonRef.current, {
        component: 'Button',
        className: 'ant-button-wave',
      });
    }

    // 执行用户点击处理
    props.onClick?.(e);
  };

  return (
    <button
      ref={composeRef(ref, buttonRef)}
      className={buttonCls}
      onClick={handleClick}
      {...restProps}
    >
      {children}
    </button>
  );
});
```

### 2. Checkbox 勾选波纹

```typescript
// components/checkbox/Checkbox.tsx 中的特殊处理
const Checkbox = React.forwardRef<HTMLInputElement, CheckboxProps>((props, ref) => {
  const handleChange = (e: ChangeEvent<HTMLInputElement>) => {
    const { checked } = e.target;

    // 只在勾选时显示波纹
    if (checked && checkboxRef.current) {
      showWaveEffect(checkboxRef.current, {
        component: 'Checkbox',
        className: 'ant-checkbox-wave',
      });
    }

    props.onChange?.(e);
  };

  return (
    <label className={wrapperCls}>
      <span className={checkboxCls} ref={checkboxRef}>
        <input
          ref={ref}
          type="checkbox"
          className="ant-checkbox-input"
          onChange={handleChange}
          {...restProps}
        />
        <span className="ant-checkbox-inner" />
      </span>
      {children}
    </label>
  );
});
```

### 3. 自定义波纹集成

```typescript
// 自定义组件中集成波纹效果
const useWaveEffect = (enabled: boolean = true) => {
  const elementRef = useRef<HTMLElement>(null);
  const [isWaving, setIsWaving] = useState(false);

  const triggerWave = useCallback((event?: React.MouseEvent) => {
    if (!enabled || !elementRef.current || isWaving) return;

    setIsWaving(true);

    showWaveEffect(elementRef.current, {
      component: 'Custom',
      className: 'custom-wave',
      onComplete: () => setIsWaving(false),
    });
  }, [enabled, isWaving]);

  return {
    elementRef,
    triggerWave,
    isWaving,
  };
};

// 使用示例
const CustomButton: React.FC = ({ children, onClick, ...props }) => {
  const { elementRef, triggerWave } = useWaveEffect();

  const handleClick = (e: React.MouseEvent) => {
    triggerWave(e);
    onClick?.(e);
  };

  return (
    <div
      ref={elementRef}
      className="custom-button"
      onClick={handleClick}
      {...props}
    >
      {children}
    </div>
  );
};
```

## 性能优化

### 1. 内存管理优化

```typescript
// 改进的内存管理策略
class OptimizedWaveManager {
  private wavePool: WaveInstance[] = [];
  private activeWaves = new WeakMap<HTMLElement, WaveInstance>();
  private maxPoolSize = 20;

  // 对象池模式，重用波纹实例
  getWaveInstance(target: HTMLElement): WaveInstance {
    let instance = this.wavePool.pop();

    if (!instance) {
      instance = new WaveInstance();
    }

    instance.initialize(target);
    this.activeWaves.set(target, instance);

    return instance;
  }

  // 回收波纹实例
  recycleWaveInstance(target: HTMLElement, instance: WaveInstance): void {
    this.activeWaves.delete(target);
    instance.reset();

    if (this.wavePool.length < this.maxPoolSize) {
      this.wavePool.push(instance);
    }
  }
}

// 优化的波纹实例，支持重用
class OptimizedWaveInstance {
  private element: HTMLElement | null = null;
  private container: HTMLElement | null = null;
  private isInitialized = false;

  initialize(element: HTMLElement): void {
    this.element = element;
    this.isInitialized = true;
    this.createContainer();
  }

  reset(): void {
    this.element = null;
    this.container?.remove();
    this.container = null;
    this.isInitialized = false;
  }

  // ... 其他方法
}
```

### 2. 动画性能优化

```typescript
// GPU 加速的波纹动画
const createOptimizedWaveAnimation = (container: HTMLElement) => {
  // 使用 transform 和 opacity，触发 GPU 加速
  const keyframes: Keyframe[] = [
    {
      transform: 'scale(0) translateZ(0)',
      opacity: 0.5,
      willChange: 'transform, opacity',
    },
    {
      transform: 'scale(1) translateZ(0)',
      opacity: 0,
      willChange: 'transform, opacity',
    }
  ];

  const options: KeyframeAnimationOptions = {
    duration: 400,
    easing: 'cubic-bezier(0.08, 0.82, 0.17, 1)',
    fill: 'forwards',
  };

  return container.animate(keyframes, options);
};

// 批量动画处理
class WaveAnimationBatch {
  private pendingAnimations: Array<{
    container: HTMLElement;
    config: WaveConfig;
  }> = [];
  private rafId: number | null = null;

  schedule(container: HTMLElement, config: WaveConfig): void {
    this.pendingAnimations.push({ container, config });

    if (!this.rafId) {
      this.rafId = requestAnimationFrame(() => this.flush());
    }
  }

  private flush(): void {
    // 批量启动动画，减少布局抖动
    for (const { container, config } of this.pendingAnimations) {
      this.startAnimation(container, config);
    }

    this.pendingAnimations = [];
    this.rafId = null;
  }

  private startAnimation(container: HTMLElement, config: WaveConfig): void {
    // 在同一帧内批量处理所有动画
    const animation = createOptimizedWaveAnimation(container);

    animation.finished.then(() => {
      // 动画完成后清理
      container.style.willChange = 'auto';
      config.onComplete?.();
    });
  }
}
```

### 3. 智能降级策略

```typescript
// 性能感知的波纹系统
class PerformanceAwareWaveManager {
  private performanceMetrics = {
    frameRate: 60,
    memoryUsage: 0,
    concurrentAnimations: 0,
  };

  private shouldUseSimplifiedWave(): boolean {
    // 检测设备性能
    const hardwareConcurrency = navigator.hardwareConcurrency || 2;
    const connection = (navigator as any).connection;

    // 低端设备或慢网络连接时简化动画
    if (hardwareConcurrency < 4) return true;
    if (connection && connection.effectiveType === 'slow-2g') return true;
    if (this.performanceMetrics.frameRate < 30) return true;
    if (this.performanceMetrics.concurrentAnimations > 5) return true;

    return false;
  }

  createWave(target: HTMLElement, options: WaveOptions): void {
    if (this.shouldUseSimplifiedWave()) {
      // 使用简化的波纹效果
      this.createSimplifiedWave(target, options);
    } else {
      // 使用完整的波纹效果
      this.createFullWave(target, options);
    }
  }

  private createSimplifiedWave(target: HTMLElement, options: WaveOptions): void {
    // 简化版：仅使用 CSS transition
    const overlay = document.createElement('div');
    overlay.className = 'ant-wave-simplified';
    overlay.style.cssText = `
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      background: rgba(24, 144, 255, 0.2);
      opacity: 0;
      transition: opacity 0.2s ease-out;
      pointer-events: none;
    `;

    target.appendChild(overlay);

    // 触发动画
    requestAnimationFrame(() => {
      overlay.style.opacity = '1';

      setTimeout(() => {
        overlay.style.opacity = '0';
        setTimeout(() => overlay.remove(), 200);
      }, 100);
    });
  }

  private createFullWave(target: HTMLElement, options: WaveOptions): void {
    // 完整版波纹效果
    showWaveEffect(target, options);
  }
}
```

## 实际应用案例

### 1. 交互式卡片波纹

```typescript
// 卡片点击的波纹反馈
const InteractiveCard: React.FC = ({ children, onClick, ...props }) => {
  const cardRef = useRef<HTMLDivElement>(null);
  const [rippleStyle, setRippleStyle] = useState<React.CSSProperties>({});

  const handleClick = (e: React.MouseEvent) => {
    if (!cardRef.current) return;

    const rect = cardRef.current.getBoundingClientRect();
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;

    // 计算波纹最大半径
    const maxRadius = Math.max(
      Math.sqrt(x ** 2 + y ** 2),
      Math.sqrt((rect.width - x) ** 2 + y ** 2),
      Math.sqrt(x ** 2 + (rect.height - y) ** 2),
      Math.sqrt((rect.width - x) ** 2 + (rect.height - y) ** 2)
    );

    // 设置波纹样式
    setRippleStyle({
      left: x - maxRadius,
      top: y - maxRadius,
      width: maxRadius * 2,
      height: maxRadius * 2,
    });

    // 触发波纹动画
    showWaveEffect(cardRef.current, {
      component: 'Card',
      className: 'card-wave',
      style: rippleStyle,
    });

    onClick?.(e);
  };

  return (
    <div
      ref={cardRef}
      className="interactive-card"
      onClick={handleClick}
      {...props}
    >
      {children}
    </div>
  );
};
```

### 2. 列表项选择波纹

```typescript
// 列表项的选择反馈
const SelectableListItem: React.FC = ({
  selected,
  onSelect,
  children,
  ...props
}) => {
  const itemRef = useRef<HTMLDivElement>(null);
  const waveTimeoutRef = useRef<NodeJS.Timeout>();

  const handleSelect = (e: React.MouseEvent) => {
    if (!itemRef.current) return;

    // 清除之前的波纹
    if (waveTimeoutRef.current) {
      clearTimeout(waveTimeoutRef.current);
    }

    // 根据选择状态使用不同的波纹效果
    const waveClass = selected ? 'list-item-wave-deselect' : 'list-item-wave-select';

    showWaveEffect(itemRef.current, {
      component: 'ListItem',
      className: waveClass,
    });

    // 延迟执行选择逻辑，让波纹先开始
    waveTimeoutRef.current = setTimeout(() => {
      onSelect?.(!selected);
    }, 50);
  };

  useEffect(() => {
    return () => {
      if (waveTimeoutRef.current) {
        clearTimeout(waveTimeoutRef.current);
      }
    };
  }, []);

  return (
    <div
      ref={itemRef}
      className={classNames('selectable-list-item', {
        'selected': selected
      })}
      onClick={handleSelect}
      {...props}
    >
      {children}
    </div>
  );
};
```

### 3. 手势触控波纹

```typescript
// 支持触控手势的波纹效果
const TouchWaveComponent: React.FC = ({ children, ...props }) => {
  const elementRef = useRef<HTMLDivElement>(null);
  const touchStartRef = useRef<{ x: number; y: number; time: number } | null>(null);

  const handleTouchStart = (e: React.TouchEvent) => {
    if (!elementRef.current || e.touches.length !== 1) return;

    const touch = e.touches[0];
    const rect = elementRef.current.getBoundingClientRect();

    touchStartRef.current = {
      x: touch.clientX - rect.left,
      y: touch.clientY - rect.top,
      time: Date.now(),
    };

    // 创建触摸反馈的小波纹
    showWaveEffect(elementRef.current, {
      component: 'Touch',
      className: 'touch-start-wave',
      scale: 0.3, // 较小的初始波纹
      duration: 200,
    });
  };

  const handleTouchEnd = (e: React.TouchEvent) => {
    if (!elementRef.current || !touchStartRef.current) return;

    const touchDuration = Date.now() - touchStartRef.current.time;

    // 根据触摸时长决定波纹效果
    if (touchDuration < 500) {
      // 短触摸：快速波纹
      showWaveEffect(elementRef.current, {
        component: 'Touch',
        className: 'touch-tap-wave',
        duration: 300,
      });
    } else {
      // 长触摸：扩散波纹
      showWaveEffect(elementRef.current, {
        component: 'Touch',
        className: 'touch-press-wave',
        duration: 600,
        scale: 1.2,
      });
    }

    touchStartRef.current = null;
  };

  return (
    <div
      ref={elementRef}
      className="touch-wave-component"
      onTouchStart={handleTouchStart}
      onTouchEnd={handleTouchEnd}
      {...props}
    >
      {children}
    </div>
  );
};
```

## 个人设计思考

### 如果让我重新设计 Wave 动画系统

#### 1. 物理引擎驱动的波纹

```typescript
// 基于物理引擎的更真实波纹效果
interface WavePhysics {
  velocity: number;      // 波纹扩散速度
  damping: number;       // 阻尼系数
  frequency: number;     // 波纹频率
  amplitude: number;     // 波纹振幅
}

class PhysicsWaveRenderer {
  private canvas: HTMLCanvasElement;
  private ctx: CanvasRenderingContext2D;
  private physics: WavePhysics;
  private particles: WaveParticle[] = [];

  constructor(container: HTMLElement, physics: WavePhysics) {
    this.canvas = document.createElement('canvas');
    this.ctx = this.canvas.getContext('2d')!;
    this.physics = physics;

    // 设置 canvas 样式
    this.canvas.style.cssText = `
      position: absolute;
      top: 0;
      left: 0;
      pointer-events: none;
      z-index: 1;
    `;

    container.appendChild(this.canvas);
    this.resizeCanvas(container);
  }

  createWave(x: number, y: number): void {
    // 创建波纹粒子
    for (let i = 0; i < 50; i++) {
      const angle = (i / 50) * Math.PI * 2;
      const particle = new WaveParticle(x, y, angle, this.physics);
      this.particles.push(particle);
    }

    this.animate();
  }

  private animate = (): void => {
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

    // 更新和渲染粒子
    this.particles = this.particles.filter(particle => {
      particle.update();
      particle.render(this.ctx);
      return !particle.isDead();
    });

    if (this.particles.length > 0) {
      requestAnimationFrame(this.animate);
    }
  };

  private resizeCanvas(container: HTMLElement): void {
    const rect = container.getBoundingClientRect();
    this.canvas.width = rect.width;
    this.canvas.height = rect.height;
  }
}

class WaveParticle {
  private x: number;
  private y: number;
  private initialX: number;
  private initialY: number;
  private angle: number;
  private distance = 0;
  private opacity = 1;
  private physics: WavePhysics;

  constructor(x: number, y: number, angle: number, physics: WavePhysics) {
    this.x = this.initialX = x;
    this.y = this.initialY = y;
    this.angle = angle;
    this.physics = physics;
  }

  update(): void {
    // 物理更新
    this.distance += this.physics.velocity;
    this.opacity = Math.max(0, 1 - (this.distance / 200) * this.physics.damping);

    // 位置更新
    this.x = this.initialX + Math.cos(this.angle) * this.distance;
    this.y = this.initialY + Math.sin(this.angle) * this.distance;
  }

  render(ctx: CanvasRenderingContext2D): void {
    const radius = this.physics.amplitude * Math.sin(this.distance * this.physics.frequency);

    ctx.beginPath();
    ctx.arc(this.x, this.y, Math.abs(radius), 0, Math.PI * 2);
    ctx.fillStyle = `rgba(24, 144, 255, ${this.opacity})`;
    ctx.fill();
  }

  isDead(): boolean {
    return this.opacity <= 0;
  }
}
```

#### 2. 手势驱动的智能波纹

```typescript
// 基于手势识别的智能波纹系统
interface GestureWaveConfig {
  tap: WaveConfig;
  press: WaveConfig;
  swipe: WaveConfig;
  pinch: WaveConfig;
}

class GestureWaveManager {
  private gestureRecognizer: GestureRecognizer;
  private waveConfigs: GestureWaveConfig;

  constructor(element: HTMLElement, configs: GestureWaveConfig) {
    this.waveConfigs = configs;
    this.gestureRecognizer = new GestureRecognizer(element);
    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    this.gestureRecognizer.on('tap', (event) => {
      this.createWave(event.center, this.waveConfigs.tap);
    });

    this.gestureRecognizer.on('press', (event) => {
      this.createWave(event.center, this.waveConfigs.press);
    });

    this.gestureRecognizer.on('swipe', (event) => {
      this.createDirectionalWave(event.start, event.end, this.waveConfigs.swipe);
    });

    this.gestureRecognizer.on('pinch', (event) => {
      this.createScaleWave(event.center, event.scale, this.waveConfigs.pinch);
    });
  }

  private createWave(center: Point, config: WaveConfig): void {
    // 标准圆形波纹
    showWaveEffect(this.element, {
      ...config,
      center,
      shape: 'circle',
    });
  }

  private createDirectionalWave(start: Point, end: Point, config: WaveConfig): void {
    // 方向性波纹，从起点向终点扩散
    const direction = Math.atan2(end.y - start.y, end.x - start.x);

    showWaveEffect(this.element, {
      ...config,
      center: start,
      shape: 'directional',
      direction,
    });
  }

  private createScaleWave(center: Point, scale: number, config: WaveConfig): void {
    // 缩放相关的波纹
    showWaveEffect(this.element, {
      ...config,
      center,
      shape: 'circle',
      scale: Math.max(0.5, Math.min(2, scale)),
    });
  }
}

// 手势识别器
class GestureRecognizer {
  private element: HTMLElement;
  private eventEmitter = new EventEmitter();
  private touches: Map<number, Touch> = new Map();
  private gestureState: GestureState = 'idle';

  constructor(element: HTMLElement) {
    this.element = element;
    this.setupTouchListeners();
  }

  on(event: string, handler: Function): void {
    this.eventEmitter.on(event, handler);
  }

  private setupTouchListeners(): void {
    this.element.addEventListener('touchstart', this.handleTouchStart.bind(this));
    this.element.addEventListener('touchmove', this.handleTouchMove.bind(this));
    this.element.addEventListener('touchend', this.handleTouchEnd.bind(this));
  }

  private handleTouchStart(e: TouchEvent): void {
    Array.from(e.changedTouches).forEach(touch => {
      this.touches.set(touch.identifier, touch);
    });

    if (this.touches.size === 1) {
      this.gestureState = 'potential-tap';
      this.startTapTimer();
    } else if (this.touches.size === 2) {
      this.gestureState = 'potential-pinch';
    }
  }

  private handleTouchMove(e: TouchEvent): void {
    // 更新触摸点
    Array.from(e.changedTouches).forEach(touch => {
      this.touches.set(touch.identifier, touch);
    });

    // 根据移动判断手势类型
    if (this.gestureState === 'potential-tap') {
      const movement = this.calculateMovement();
      if (movement > 10) {
        this.gestureState = 'swipe';
      }
    }
  }

  private handleTouchEnd(e: TouchEvent): void {
    Array.from(e.changedTouches).forEach(touch => {
      this.touches.delete(touch.identifier);
    });

    // 触发相应的手势事件
    this.processGesture();
    this.reset();
  }

  private processGesture(): void {
    switch (this.gestureState) {
      case 'potential-tap':
        this.eventEmitter.emit('tap', {
          center: this.getTouchCenter(),
        });
        break;
      case 'press':
        this.eventEmitter.emit('press', {
          center: this.getTouchCenter(),
        });
        break;
      case 'swipe':
        this.eventEmitter.emit('swipe', {
          start: this.getSwipeStart(),
          end: this.getSwipeEnd(),
        });
        break;
      case 'pinch':
        this.eventEmitter.emit('pinch', {
          center: this.getTouchCenter(),
          scale: this.getPinchScale(),
        });
        break;
    }
  }

  // ... 其他辅助方法
}
```

#### 3. 可视化波纹编辑器

```typescript
// 波纹效果的可视化编辑工具
interface WavePreset {
  id: string;
  name: string;
  config: WaveConfig;
  preview: string; // base64 编码的预览图
}

class WaveEditor {
  private canvas: HTMLCanvasElement;
  private previewContainer: HTMLElement;
  private controlPanel: HTMLElement;
  private currentConfig: WaveConfig;

  constructor(container: HTMLElement) {
    this.setupUI(container);
    this.bindEvents();
    this.loadPresets();
  }

  private setupUI(container: HTMLElement): void {
    container.innerHTML = `
      <div class="wave-editor">
        <div class="wave-preview">
          <canvas id="wave-preview-canvas"></canvas>
          <button id="preview-trigger">预览波纹</button>
        </div>
        <div class="wave-controls">
          <div class="control-group">
            <label>持续时间</label>
            <input type="range" id="duration" min="100" max="1000" value="400">
            <span id="duration-value">400ms</span>
          </div>
          <div class="control-group">
            <label>扩散速度</label>
            <input type="range" id="velocity" min="0.5" max="3" step="0.1" value="1">
            <span id="velocity-value">1x</span>
          </div>
          <div class="control-group">
            <label>透明度</label>
            <input type="range" id="opacity" min="0.1" max="1" step="0.1" value="0.5">
            <span id="opacity-value">0.5</span>
          </div>
          <div class="control-group">
            <label>颜色</label>
            <input type="color" id="color" value="#1677ff">
          </div>
          <div class="control-group">
            <label>缓动函数</label>
            <select id="easing">
              <option value="linear">Linear</option>
              <option value="ease">Ease</option>
              <option value="ease-in">Ease In</option>
              <option value="ease-out" selected>Ease Out</option>
              <option value="ease-in-out">Ease In Out</option>
            </select>
          </div>
        </div>
        <div class="wave-presets">
          <h3>预设效果</h3>
          <div id="preset-list"></div>
          <button id="save-preset">保存当前配置</button>
        </div>
        <div class="wave-export">
          <button id="export-config">导出配置</button>
          <button id="export-css">导出 CSS</button>
        </div>
      </div>
    `;

    this.canvas = container.querySelector('#wave-preview-canvas') as HTMLCanvasElement;
    this.previewContainer = container.querySelector('.wave-preview') as HTMLElement;
    this.controlPanel = container.querySelector('.wave-controls') as HTMLElement;
  }

  private bindEvents(): void {
    // 控制面板事件
    this.controlPanel.addEventListener('input', (e) => {
      this.updateConfig();
      this.updatePreview();
    });

    // 预览按钮
    const previewBtn = document.getElementById('preview-trigger');
    previewBtn?.addEventListener('click', () => {
      this.triggerPreview();
    });

    // 导出按钮
    document.getElementById('export-config')?.addEventListener('click', () => {
      this.exportConfig();
    });

    document.getElementById('export-css')?.addEventListener('click', () => {
      this.exportCSS();
    });
  }

  private updateConfig(): void {
    this.currentConfig = {
      duration: parseInt((document.getElementById('duration') as HTMLInputElement).value),
      velocity: parseFloat((document.getElementById('velocity') as HTMLInputElement).value),
      opacity: parseFloat((document.getElementById('opacity') as HTMLInputElement).value),
      color: (document.getElementById('color') as HTMLInputElement).value,
      easing: (document.getElementById('easing') as HTMLSelectElement).value,
    };
  }

  private triggerPreview(): void {
    // 在预览容器中创建波纹效果
    const rect = this.previewContainer.getBoundingClientRect();
    const centerX = rect.width / 2;
    const centerY = rect.height / 2;

    this.createWaveAt(centerX, centerY, this.currentConfig);
  }

  private createWaveAt(x: number, y: number, config: WaveConfig): void {
    const waveElement = document.createElement('div');
    waveElement.className = 'preview-wave';
    waveElement.style.cssText = `
      position: absolute;
      left: ${x}px;
      top: ${y}px;
      width: 0;
      height: 0;
      border-radius: 50%;
      background: ${config.color};
      opacity: ${config.opacity};
      transform: translate(-50%, -50%);
      pointer-events: none;
    `;

    this.previewContainer.appendChild(waveElement);

    // 启动动画
    const animation = waveElement.animate([
      {
        width: '0px',
        height: '0px',
        opacity: config.opacity,
      },
      {
        width: '200px',
        height: '200px',
        opacity: 0,
      }
    ], {
      duration: config.duration,
      easing: config.easing,
      fill: 'forwards',
    });

    animation.finished.then(() => {
      waveElement.remove();
    });
  }

  private exportConfig(): void {
    const configJson = JSON.stringify(this.currentConfig, null, 2);
    this.downloadFile('wave-config.json', configJson);
  }

  private exportCSS(): void {
    const css = this.generateCSS(this.currentConfig);
    this.downloadFile('wave-animation.css', css);
  }

  private generateCSS(config: WaveConfig): string {
    return `
      @keyframes waveEffect {
        0% {
          transform: scale(0);
          opacity: ${config.opacity};
        }
        100% {
          transform: scale(1);
          opacity: 0;
        }
      }

      .wave-animation {
        animation: waveEffect ${config.duration}ms ${config.easing} forwards;
        background: radial-gradient(circle, ${config.color} 10%, transparent 10.01%);
        border-radius: 50%;
        position: absolute;
        pointer-events: none;
      }
    `;
  }

  private downloadFile(filename: string, content: string): void {
    const blob = new Blob([content], { type: 'text/plain' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    a.click();
    URL.revokeObjectURL(url);
  }
}
```

## 优化建议

### 1. 当前实现的改进空间

#### 改进点 1：波纹形状自适应
```typescript
// 根据元素形状智能调整波纹
const calculateOptimalWaveShape = (element: HTMLElement): WaveShape => {
  const computedStyle = getComputedStyle(element);
  const rect = element.getBoundingClientRect();

  // 分析元素形状特征
  const borderRadius = parseFloat(computedStyle.borderRadius) || 0;
  const aspectRatio = rect.width / rect.height;

  if (borderRadius > Math.min(rect.width, rect.height) / 4) {
    // 圆角较大，使用圆形波纹
    return {
      type: 'circle',
      radius: Math.max(rect.width, rect.height) / 2,
    };
  } else if (Math.abs(aspectRatio - 1) < 0.2) {
    // 接近正方形，使用同心方形波纹
    return {
      type: 'square',
      size: Math.max(rect.width, rect.height),
    };
  } else {
    // 矩形，使用椭圆波纹
    return {
      type: 'ellipse',
      width: rect.width,
      height: rect.height,
    };
  }
};

// 创建自适应波纹
const createAdaptiveWave = (element: HTMLElement, clickPoint: Point) => {
  const shape = calculateOptimalWaveShape(element);

  switch (shape.type) {
    case 'circle':
      return createCircularWave(element, clickPoint, shape.radius);
    case 'square':
      return createSquareWave(element, clickPoint, shape.size);
    case 'ellipse':
      return createEllipticalWave(element, clickPoint, shape.width, shape.height);
  }
};
```

#### 改进点 2：多点触控波纹
```typescript
// 支持多点触控的波纹效果
class MultiTouchWaveManager {
  private activeTouches = new Map<number, TouchWave>();
  private element: HTMLElement;

  constructor(element: HTMLElement) {
    this.element = element;
    this.setupEventListeners();
  }

  private setupEventListeners(): void {
    this.element.addEventListener('touchstart', this.handleTouchStart.bind(this));
    this.element.addEventListener('touchmove', this.handleTouchMove.bind(this));
    this.element.addEventListener('touchend', this.handleTouchEnd.bind(this));
  }

  private handleTouchStart(e: TouchEvent): void {
    Array.from(e.changedTouches).forEach(touch => {
      const rect = this.element.getBoundingClientRect();
      const waveConfig = {
        x: touch.clientX - rect.left,
        y: touch.clientY - rect.top,
        identifier: touch.identifier,
      };

      const touchWave = new TouchWave(this.element, waveConfig);
      this.activeTouches.set(touch.identifier, touchWave);
      touchWave.start();
    });
  }

  private handleTouchMove(e: TouchEvent): void {
    // 更新波纹位置（用于拖拽效果）
    Array.from(e.changedTouches).forEach(touch => {
      const touchWave = this.activeTouches.get(touch.identifier);
      if (touchWave) {
        const rect = this.element.getBoundingClientRect();
        touchWave.updatePosition(
          touch.clientX - rect.left,
          touch.clientY - rect.top
        );
      }
    });
  }

  private handleTouchEnd(e: TouchEvent): void {
    Array.from(e.changedTouches).forEach(touch => {
      const touchWave = this.activeTouches.get(touch.identifier);
      if (touchWave) {
        touchWave.finish();
        this.activeTouches.delete(touch.identifier);
      }
    });
  }
}

class TouchWave {
  private element: HTMLElement;
  private config: TouchWaveConfig;
  private waveElement: HTMLElement;
  private animation: Animation | null = null;

  constructor(element: HTMLElement, config: TouchWaveConfig) {
    this.element = element;
    this.config = config;
    this.createWaveElement();
  }

  start(): void {
    this.animation = this.waveElement.animate([
      { transform: 'scale(0)', opacity: '0.6' },
      { transform: 'scale(1)', opacity: '0' }
    ], {
      duration: 600,
      easing: 'cubic-bezier(0.4, 0, 0.2, 1)',
      fill: 'forwards'
    });
  }

  updatePosition(x: number, y: number): void {
    // 更新波纹中心位置
    this.waveElement.style.left = `${x}px`;
    this.waveElement.style.top = `${y}px`;
  }

  finish(): void {
    // 加速完成动画
    if (this.animation) {
      this.animation.playbackRate = 2;
    }

    setTimeout(() => this.cleanup(), 300);
  }

  private createWaveElement(): void {
    this.waveElement = document.createElement('div');
    this.waveElement.className = 'multi-touch-wave';
    this.waveElement.style.cssText = `
      position: absolute;
      left: ${this.config.x}px;
      top: ${this.config.y}px;
      width: 100px;
      height: 100px;
      border-radius: 50%;
      background: radial-gradient(circle, rgba(24, 144, 255, 0.3) 0%, transparent 70%);
      transform: translate(-50%, -50%) scale(0);
      pointer-events: none;
    `;

    this.element.appendChild(this.waveElement);
  }

  private cleanup(): void {
    this.waveElement.remove();
    this.animation = null;
  }
}
```

### 2. 新增功能建议

#### 建议 1：声音同步波纹
```typescript
// 与音频同步的波纹效果
class AudioSyncWaveManager {
  private audioContext: AudioContext;
  private analyser: AnalyserNode;
  private dataArray: Uint8Array;
  private element: HTMLElement;
  private waveRenderer: AudioWaveRenderer;

  constructor(element: HTMLElement, audioElement: HTMLAudioElement) {
    this.element = element;
    this.setupAudioAnalysis(audioElement);
    this.waveRenderer = new AudioWaveRenderer(element);
  }

  private setupAudioAnalysis(audioElement: HTMLAudioElement): void {
    this.audioContext = new AudioContext();
    const source = this.audioContext.createMediaElementSource(audioElement);
    this.analyser = this.audioContext.createAnalyser();

    source.connect(this.analyser);
    this.analyser.connect(this.audioContext.destination);

    this.analyser.fftSize = 256;
    this.dataArray = new Uint8Array(this.analyser.frequencyBinCount);
  }

  startVisualization(): void {
    const animate = () => {
      this.analyser.getByteFrequencyData(this.dataArray);

      // 分析音频特征
      const bass = this.getFrequencyRange(0, 60);      // 低频
      const mid = this.getFrequencyRange(60, 120);     // 中频
      const treble = this.getFrequencyRange(120, 256); // 高频

      // 根据音频特征生成波纹
      this.waveRenderer.render({
        bass: bass / 255,
        mid: mid / 255,
        treble: treble / 255,
      });

      requestAnimationFrame(animate);
    };

    animate();
  }

  private getFrequencyRange(start: number, end: number): number {
    let sum = 0;
    for (let i = start; i < Math.min(end, this.dataArray.length); i++) {
      sum += this.dataArray[i];
    }
    return sum / (end - start);
  }
}

class AudioWaveRenderer {
  private canvas: HTMLCanvasElement;
  private ctx: CanvasRenderingContext2D;
  private particles: AudioParticle[] = [];

  constructor(element: HTMLElement) {
    this.canvas = document.createElement('canvas');
    this.ctx = this.canvas.getContext('2d')!;
    this.setupCanvas(element);
  }

  render(audioData: AudioData): void {
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

    // 根据音频数据创建新粒子
    if (audioData.bass > 0.5) {
      this.createBassWave();
    }
    if (audioData.mid > 0.3) {
      this.createMidWave();
    }
    if (audioData.treble > 0.4) {
      this.createTrebleWave();
    }

    // 更新和渲染粒子
    this.particles = this.particles.filter(particle => {
      particle.update(audioData);
      particle.render(this.ctx);
      return !particle.isDead();
    });
  }

  private createBassWave(): void {
    // 低频创建大而慢的波纹
    const particle = new AudioParticle({
      x: this.canvas.width / 2,
      y: this.canvas.height / 2,
      radius: 50,
      color: 'rgba(255, 0, 0, 0.3)',
      speed: 2,
      lifespan: 100,
    });
    this.particles.push(particle);
  }

  private createMidWave(): void {
    // 中频创建中等波纹
    const particle = new AudioParticle({
      x: Math.random() * this.canvas.width,
      y: Math.random() * this.canvas.height,
      radius: 30,
      color: 'rgba(0, 255, 0, 0.3)',
      speed: 3,
      lifespan: 60,
    });
    this.particles.push(particle);
  }

  private createTrebleWave(): void {
    // 高频创建小而快的波纹
    for (let i = 0; i < 3; i++) {
      const particle = new AudioParticle({
        x: Math.random() * this.canvas.width,
        y: Math.random() * this.canvas.height,
        radius: 10,
        color: 'rgba(0, 0, 255, 0.3)',
        speed: 5,
        lifespan: 30,
      });
      this.particles.push(particle);
    }
  }
}
```

#### 建议 2：情感化波纹效果
```typescript
// 基于用户情感状态的波纹效果
interface EmotionWaveConfig {
  happiness: WaveStyle;
  sadness: WaveStyle;
  anger: WaveStyle;
  surprise: WaveStyle;
  fear: WaveStyle;
  neutral: WaveStyle;
}

class EmotionalWaveManager {
  private currentEmotion: EmotionType = 'neutral';
  private emotionConfigs: EmotionWaveConfig;
  private element: HTMLElement;

  constructor(element: HTMLElement, configs: EmotionWaveConfig) {
    this.element = element;
    this.emotionConfigs = configs;
  }

  setEmotion(emotion: EmotionType): void {
    this.currentEmotion = emotion;
  }

  createEmotionalWave(x: number, y: number): void {
    const config = this.emotionConfigs[this.currentEmotion];

    switch (this.currentEmotion) {
      case 'happiness':
        this.createHappyWave(x, y, config);
        break;
      case 'sadness':
        this.createSadWave(x, y, config);
        break;
      case 'anger':
        this.createAngryWave(x, y, config);
        break;
      case 'surprise':
        this.createSurpriseWave(x, y, config);
        break;
      case 'fear':
        this.createFearWave(x, y, config);
        break;
      default:
        this.createNeutralWave(x, y, config);
    }
  }

  private createHappyWave(x: number, y: number, config: WaveStyle): void {
    // 快乐：明亮的颜色，快速扩散，多个小波纹
    for (let i = 0; i < 5; i++) {
      setTimeout(() => {
        this.createSingleWave(x, y, {
          ...config,
          color: `hsl(${Math.random() * 60 + 30}, 80%, 60%)`, // 暖色调
          duration: 300,
          scale: 0.8 + Math.random() * 0.4,
        });
      }, i * 50);
    }
  }

  private createSadWave(x: number, y: number, config: WaveStyle): void {
    // 悲伤：冷色调，缓慢扩散，渐变消失
    this.createSingleWave(x, y, {
      ...config,
      color: 'rgba(100, 150, 200, 0.3)',
      duration: 800,
      easing: 'ease-out',
      pattern: 'fadeDown', // 向下渐变消失
    });
  }

  private createAngryWave(x: number, y: number, config: WaveStyle): void {
    // 愤怒：红色，尖锐扩散，边缘锯齿
    this.createSingleWave(x, y, {
      ...config,
      color: 'rgba(255, 50, 50, 0.6)',
      duration: 200,
      easing: 'ease-in',
      shape: 'jagged', // 锯齿边缘
    });
  }

  private createSurpriseWave(x: number, y: number, config: WaveStyle): void {
    // 惊讶：突然爆发，快速扩散后收缩
    this.createSingleWave(x, y, {
      ...config,
      color: 'rgba(255, 255, 0, 0.5)',
      duration: 400,
      pattern: 'explode-contract', // 爆发后收缩
    });
  }

  private createFearWave(x: number, y: number, config: WaveStyle): void {
    // 恐惧：颤抖效果，不规则扩散
    this.createSingleWave(x, y, {
      ...config,
      color: 'rgba(120, 0, 120, 0.4)',
      duration: 600,
      pattern: 'trembling', // 颤抖效果
    });
  }

  private createNeutralWave(x: number, y: number, config: WaveStyle): void {
    // 中性：标准波纹
    this.createSingleWave(x, y, config);
  }

  private createSingleWave(x: number, y: number, style: WaveStyle): void {
    const waveElement = document.createElement('div');
    waveElement.className = `emotion-wave emotion-wave-${this.currentEmotion}`;

    // 应用样式和动画
    this.applyWaveStyle(waveElement, x, y, style);
    this.element.appendChild(waveElement);

    // 启动动画
    this.startWaveAnimation(waveElement, style);
  }

  private applyWaveStyle(element: HTMLElement, x: number, y: number, style: WaveStyle): void {
    element.style.cssText = `
      position: absolute;
      left: ${x}px;
      top: ${y}px;
      width: 20px;
      height: 20px;
      border-radius: 50%;
      background: ${style.color};
      transform: translate(-50%, -50%) scale(0);
      pointer-events: none;
    `;

    // 应用特殊形状
    if (style.shape === 'jagged') {
      element.style.clipPath = 'polygon(50% 0%, 60% 35%, 98% 35%, 68% 57%, 79% 91%, 50% 70%, 21% 91%, 32% 57%, 2% 35%, 40% 35%)';
    }
  }

  private startWaveAnimation(element: HTMLElement, style: WaveStyle): void {
    let keyframes: Keyframe[];

    switch (style.pattern) {
      case 'fadeDown':
        keyframes = [
          { transform: 'translate(-50%, -50%) scale(0)', opacity: '0.6' },
          { transform: 'translate(-50%, 0%) scale(1.5)', opacity: '0' }
        ];
        break;
      case 'explode-contract':
        keyframes = [
          { transform: 'translate(-50%, -50%) scale(0)', opacity: '0.8' },
          { transform: 'translate(-50%, -50%) scale(2)', opacity: '0.4', offset: 0.3 },
          { transform: 'translate(-50%, -50%) scale(0.5)', opacity: '0' }
        ];
        break;
      case 'trembling':
        keyframes = [
          { transform: 'translate(-50%, -50%) scale(0)', opacity: '0.5' },
          { transform: 'translate(-45%, -55%) scale(0.8)', opacity: '0.3', offset: 0.3 },
          { transform: 'translate(-55%, -45%) scale(1.2)', opacity: '0.2', offset: 0.6 },
          { transform: 'translate(-50%, -50%) scale(1.5)', opacity: '0' }
        ];
        break;
      default:
        keyframes = [
          { transform: 'translate(-50%, -50%) scale(0)', opacity: '0.5' },
          { transform: 'translate(-50%, -50%) scale(1)', opacity: '0' }
        ];
    }

    const animation = element.animate(keyframes, {
      duration: style.duration || 400,
      easing: style.easing || 'ease-out',
      fill: 'forwards'
    });

    animation.finished.then(() => {
      element.remove();
    });
  }
}
```

## 潜在问题

### 1. 性能相关问题

#### 问题 1：频繁的 DOM 操作
```typescript
// 当前问题：每个波纹都创建新的 DOM 元素

// 改进方案：使用 Canvas 或 Web GL 渲染
class CanvasWaveRenderer {
  private canvas: HTMLCanvasElement;
  private ctx: CanvasRenderingContext2D;
  private waves: CanvasWave[] = [];
  private animationId: number | null = null;

  constructor(container: HTMLElement) {
    this.canvas = document.createElement('canvas');
    this.ctx = this.canvas.getContext('2d')!;
    this.setupCanvas(container);
  }

  createWave(x: number, y: number, config: WaveConfig): void {
    const wave = new CanvasWave(x, y, config);
    this.waves.push(wave);

    if (!this.animationId) {
      this.startAnimation();
    }
  }

  private startAnimation(): void {
    const animate = () => {
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

      // 更新和渲染所有波纹
      this.waves = this.waves.filter(wave => {
        wave.update();
        wave.render(this.ctx);
        return !wave.isComplete();
      });

      if (this.waves.length > 0) {
        this.animationId = requestAnimationFrame(animate);
      } else {
        this.animationId = null;
      }
    };

    this.animationId = requestAnimationFrame(animate);
  }
}

class CanvasWave {
  private x: number;
  private y: number;
  private radius = 0;
  private opacity = 0.5;
  private config: WaveConfig;
  private startTime: number;

  constructor(x: number, y: number, config: WaveConfig) {
    this.x = x;
    this.y = y;
    this.config = config;
    this.startTime = Date.now();
  }

  update(): void {
    const elapsed = Date.now() - this.startTime;
    const progress = Math.min(elapsed / this.config.duration, 1);

    // 使用缓动函数计算当前状态
    const easedProgress = this.easeOut(progress);

    this.radius = easedProgress * this.config.maxRadius;
    this.opacity = (1 - progress) * this.config.initialOpacity;
  }

  render(ctx: CanvasRenderingContext2D): void {
    ctx.save();
    ctx.globalAlpha = this.opacity;
    ctx.fillStyle = this.config.color;

    // 创建渐变
    const gradient = ctx.createRadialGradient(
      this.x, this.y, 0,
      this.x, this.y, this.radius
    );
    gradient.addColorStop(0, this.config.color);
    gradient.addColorStop(1, 'transparent');

    ctx.fillStyle = gradient;
    ctx.beginPath();
    ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
    ctx.fill();
    ctx.restore();
  }

  isComplete(): boolean {
    return Date.now() - this.startTime >= this.config.duration;
  }

  private easeOut(t: number): number {
    return 1 - Math.pow(1 - t, 3);
  }
}
```

#### 问题 2：内存泄漏风险
```typescript
// 当前问题：波纹元素可能没有正确清理

// 改进方案：完善的生命周期管理
class WaveLifecycleManager {
  private activeWaves = new Set<WaveInstance>();
  private cleanupQueue = new Set<WaveInstance>();
  private maxWaves = 50;
  private cleanupInterval: NodeJS.Timeout;

  constructor() {
    // 定期清理过期的波纹
    this.cleanupInterval = setInterval(this.performCleanup.bind(this), 1000);
  }

  createWave(element: HTMLElement, config: WaveConfig): WaveInstance {
    // 限制并发波纹数量
    if (this.activeWaves.size >= this.maxWaves) {
      this.forceCleanupOldest();
    }

    const wave = new ManagedWaveInstance(element, config);
    this.activeWaves.add(wave);

    // 注册清理回调
    wave.onComplete(() => {
      this.scheduleWaveCleanup(wave);
    });

    wave.start();
    return wave;
  }

  private scheduleWaveCleanup(wave: WaveInstance): void {
    this.cleanupQueue.add(wave);
  }

  private performCleanup(): void {
    for (const wave of this.cleanupQueue) {
      wave.destroy();
      this.activeWaves.delete(wave);
      this.cleanupQueue.delete(wave);
    }
  }

  private forceCleanupOldest(): void {
    // 强制清理最旧的波纹
    const oldestWave = this.activeWaves.values().next().value;
    if (oldestWave) {
      oldestWave.destroy();
      this.activeWaves.delete(oldestWave);
    }
  }

  destroy(): void {
    // 清理所有资源
    for (const wave of this.activeWaves) {
      wave.destroy();
    }
    this.activeWaves.clear();
    this.cleanupQueue.clear();
    clearInterval(this.cleanupInterval);
  }
}

class ManagedWaveInstance {
  private element: HTMLElement;
  private config: WaveConfig;
  private waveElement: HTMLElement | null = null;
  private animation: Animation | null = null;
  private isDestroyed = false;
  private completionCallbacks: Array<() => void> = [];

  constructor(element: HTMLElement, config: WaveConfig) {
    this.element = element;
    this.config = config;
  }

  onComplete(callback: () => void): void {
    this.completionCallbacks.push(callback);
  }

  start(): void {
    if (this.isDestroyed) return;

    this.createWaveElement();
    this.startAnimation();
  }

  destroy(): void {
    if (this.isDestroyed) return;

    this.isDestroyed = true;
    this.animation?.cancel();
    this.waveElement?.remove();

    // 调用完成回调
    this.completionCallbacks.forEach(callback => callback());
    this.completionCallbacks = [];
  }

  private createWaveElement(): void {
    // 创建波纹元素的逻辑
  }

  private startAnimation(): void {
    // 启动动画的逻辑
  }
}
```

### 2. 用户体验问题

#### 问题 1：过度动画导致的干扰
```typescript
// 问题：频繁的波纹动画可能干扰用户

// 改进方案：智能动画频率控制
class WaveFrequencyController {
  private lastWaveTime = 0;
  private waveCount = 0;
  private windowStart = 0;
  private readonly minInterval = 100; // 最小间隔 100ms
  private readonly maxWavesPerSecond = 5;

  shouldCreateWave(): boolean {
    const now = Date.now();

    // 检查最小间隔
    if (now - this.lastWaveTime < this.minInterval) {
      return false;
    }

    // 检查频率限制
    if (now - this.windowStart > 1000) {
      // 重置计数窗口
      this.windowStart = now;
      this.waveCount = 0;
    }

    if (this.waveCount >= this.maxWavesPerSecond) {
      return false;
    }

    this.lastWaveTime = now;
    this.waveCount++;
    return true;
  }
}

// 集成到波纹管理器
class ThrottledWaveManager {
  private frequencyController = new WaveFrequencyController();
  private pendingWaves: Array<() => void> = [];

  createWave(element: HTMLElement, config: WaveConfig): void {
    const waveFactory = () => {
      if (this.frequencyController.shouldCreateWave()) {
        showWaveEffect(element, config);
      } else {
        // 将波纹加入队列
        this.pendingWaves.push(waveFactory);
        this.processPendingWaves();
      }
    };

    waveFactory();
  }

  private processPendingWaves(): void {
    if (this.pendingWaves.length === 0) return;

    setTimeout(() => {
      const nextWave = this.pendingWaves.shift();
      if (nextWave) {
        nextWave();
      }
    }, this.frequencyController.minInterval);
  }
}
```

#### 问题 2：可访问性支持不足
```typescript
// 问题：缺少对辅助技术的支持

// 改进方案：可访问性增强
class AccessibleWaveManager {
  private shouldReduceMotion = false;
  private element: HTMLElement;

  constructor(element: HTMLElement) {
    this.element = element;
    this.checkMotionPreferences();
    this.setupAccessibilityFeatures();
  }

  private checkMotionPreferences(): void {
    // 检查用户动画偏好
    const mediaQuery = window.matchMedia('(prefers-reduced-motion: reduce)');
    this.shouldReduceMotion = mediaQuery.matches;

    mediaQuery.addEventListener('change', (e) => {
      this.shouldReduceMotion = e.matches;
    });
  }

  private setupAccessibilityFeatures(): void {
    // 为屏幕阅读器添加 ARIA 属性
    this.element.setAttribute('role', 'button');
    this.element.setAttribute('aria-describedby', 'wave-feedback-description');

    // 创建隐藏的描述文本
    const description = document.createElement('div');
    description.id = 'wave-feedback-description';
    description.className = 'sr-only';
    description.textContent = '点击时会有视觉反馈效果';
    document.body.appendChild(description);
  }

  createWave(x: number, y: number, config: WaveConfig): void {
    if (this.shouldReduceMotion) {
      // 减少动画：使用简单的高亮效果
      this.createReducedMotionFeedback();
    } else {
      // 正常动画
      this.createNormalWave(x, y, config);
    }

    // 为屏幕阅读器提供反馈
    this.announceWaveAction();
  }

  private createReducedMotionFeedback(): void {
    this.element.classList.add('wave-feedback-reduced');
    setTimeout(() => {
      this.element.classList.remove('wave-feedback-reduced');
    }, 200);
  }

  private announceWaveAction(): void {
    // 使用 ARIA live region 通知屏幕阅读器
    const announcement = document.createElement('div');
    announcement.setAttribute('aria-live', 'polite');
    announcement.className = 'sr-only';
    announcement.textContent = '操作已确认';

    document.body.appendChild(announcement);
    setTimeout(() => {
      document.body.removeChild(announcement);
    }, 1000);
  }
}

// 对应的 CSS
const accessibilityCSS = `
  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    border: 0;
  }

  .wave-feedback-reduced {
    background-color: rgba(24, 144, 255, 0.1);
    transition: background-color 0.2s ease;
  }

  @media (prefers-reduced-motion: reduce) {
    .wave-motion-appear {
      animation: none !important;
    }
  }
`;
```

## 学习价值

### 1. 交互设计原则

**即时反馈**：
- 波纹动画提供点击的即时视觉确认
- 帮助用户理解操作已被系统接收
- 增强用户对界面响应性的感知

**物理隐喻**：
- 模拟真实世界的水波纹扩散
- 符合用户的物理世界经验
- 提供直观的视觉语言

**情感连接**：
- 优雅的动画增加产品的愉悦感
- 减少操作的机械感
- 提升用户的情感参与度

### 2. 技术实现策略

**性能优化技巧**：
- 使用 `transform` 和 `opacity` 属性实现 GPU 加速
- 动画结束后及时清理 DOM 元素
- 合理控制并发动画数量

**内存管理模式**：
- 对象池模式重用动画实例
- WeakMap 存储元素相关数据
- 定时清理过期资源

**响应式设计**：
- 根据设备性能调整动画复杂度
- 支持 `prefers-reduced-motion` 媒体查询
- 提供多级降级策略

### 3. 系统架构设计

**分层架构**：
- 清晰的触发、计算、渲染、清理层次
- 职责分离便于维护和扩展
- 统一的接口设计

**事件驱动模式**：
- 基于用户交互触发动画
- 生命周期事件管理
- 异步处理提升性能

**可扩展性设计**：
- 插件化的动画类型
- 配置驱动的效果定制
- 开放的 API 接口

### 4. 用户体验设计

**可访问性考虑**：
- 遵循 WCAG 无障碍标准
- 提供动画偏好设置支持
- 屏幕阅读器友好

**情感化设计**：
- 不同场景使用不同的波纹效果
- 颜色和速度传达情感信息
- 增强用户的情感体验

**性能感知**：
- 流畅的动画提升性能感知
- 合理的动画时长避免等待感
- 智能降级保证基础功能

## 总结

Wave 波纹动画作为 Ant Design 5 动画系统中最具交互性的组件，展现了以下核心价值：

### 技术创新
1. **动态计算**：实时计算元素尺寸、位置、颜色等属性
2. **性能优化**：GPU 加速、对象池、智能清理等优化策略
3. **响应式适配**：根据设备性能和用户偏好自适应调整

### 设计智慧
1. **物理仿真**：模拟真实水波纹的扩散效果
2. **情感化交互**：通过视觉反馈增强用户的操作确认感
3. **细节打磨**：边框圆角适配、颜色自动提取等细节处理

### 应用价值
1. **提升交互体验**：为用户操作提供即时、直观的视觉反馈
2. **增强品牌感知**：精致的动画效果提升产品品质感
3. **减少认知负担**：视觉反馈帮助用户确认操作状态

这套 Wave 动画系统为现代 Web 应用的交互设计提供了优秀的参考实现，特别是在平衡视觉效果、性能表现和可访问性方面的处理，值得深入研究和借鉴应用。