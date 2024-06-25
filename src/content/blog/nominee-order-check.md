---
title: 'Nominee Order Check'
description: 'Nominee Order Check.'
pubDate: 'Jul 01 2024'
tags:
  - Go
badge: 还没写好
---

## 阅读下面这段代码：

```go
func (s *wealthAdminService) NomineeOrderConfirmPreCheck(ctx *fiber.Ctx, r *mapping.NomineeOrderConfirmPreCheckReqParams) error {
	l := log.FromContext(ctx.UserContext())
	userContext := ctx.UserContext()
	resp := mapping.NomineeOrderConfirmPreCheckResult{Orders: mapping.NomineeOrderConfirmPreCheckResultOrders{},
		Config: mapping.NomineeOrderConfirmPreCheckResultConfig{
			Redeem: mapping.NomineeOrderConfirmPreCheckResultConfigRedeem{
				QuantityDifferenceUpperLimit: s.Config.GetString("wealthAdmin.redeemQuantityDifferenceUpperLimit"),
				QuantityDifferenceLowerLimit: s.Config.GetString("wealthAdmin.redeemQuantityDifferenceLowerLimit"),
			},
		}}

	m, err := s.dbAccess.GetWealthNomineeOrderById(userContext, r.Params.OrderId)
	if err != nil {
		if errors.Is(err, gorm.ErrRecordNotFound) {
			l.Info("wealth nominee order not found", zap.Error(err), zap.Int("nomineeOrderId", r.Params.OrderId))
			return code.Response(ctx, code.NewDefaultError(code.ErrNotFound), nil)
		}

		l.Error("query nominee wealth order failed", zap.Error(err), zap.Int("nomineeOrderId", r.Params.OrderId))
		return code.Response(ctx, code.NewDefaultError(code.ErrDatabase), nil)
	}

	// 确认份额上一个状态必须是上手已受理
	if !mapping.CheckNomineeOrderChangeAllow(r.Params.Ops.OrderStatus, m.OrderStatus) {
		return code.Response(ctx, code.NewError(code.ErrInvalidBody, "last order status must be accept", nil), nil)
	}

	err = copier.CopyWithOption(&resp.NomineeOrder, m, copier.Option{IgnoreEmpty: true})
	if err != nil {
		l.Info("copy database model to resp model failed", zap.Error(err))
		return code.Response(ctx, code.NewDefaultError(code.ErrBuildResp), nil)
	}

	err = copier.CopyWithOption(&resp.NomineeOrder, r.Params.Ops, copier.Option{IgnoreEmpty: true})
	if err != nil {
		l.Info("copy req model to resp model failed", zap.Error(err))
		return code.Response(ctx, code.NewDefaultError(code.ErrBuildResp), nil)
	}

	var precision int32 // 默认份额精度

	// 查找指定产品的小数位
	// 查找指定产品的小数位
	channel := utils.GenerateChannel(ctx.Get(constant.HeaderXSource))
	params := mmapping.ProductBasicInfoParams{
		ProductCode: m.ProductCode,
		Channel:     channel,
	}
	if f, err := s.MKTWealthClient.GetProductBasicInfo(ctx.UserContext(), params); err != nil {
		l.Warn("get fund product info failed", zap.Error(err),
			zap.Int("orderId", m.Id), zap.String("productCode", m.ProductCode))
	} else {
		precision = f.GetPrecision()
	}

	orders, err := s.dbAccess.GetWealthOrdersByNomineeId(ctx.UserContext(), nil, m.Id, "applyAmount desc, submitTime asc") // 按照创建时间倒序
	if err != nil {
		l.Error("search wealth orders failed", zap.Error(err), zap.Int("nomineeOrderId", r.Params.OrderId))
		return code.Response(ctx, code.NewDefaultError(code.ErrUnknow), nil)
	}

	if len(orders) == 0 {
		return code.Response(ctx, nil, resp)
	}

	// 拆单计算
	// 认购计算比例用的是认购金额
	// 赎回计算比例用的是赎回份额
	var result map[string][]decimal.Decimal
	switch m.Direction {
	case orderconstant.W_TRX_DIR_BUY:
		orderApplyAmounts, err := wealth.TWealthOrders(orders).ApplyAmounts()
		if err != nil {
			l.Info("collect orders applyAmount failed", zap.Error(err))
			return code.Response(ctx, code.NewDefaultError(code.ErrUnknow), nil)
		}

		result = quantitycalcuator.AllocateGetQuantityByAmount(orderApplyAmounts,
			r.Params.Ops.FilledAmountDecimal(), r.Params.Ops.FilledQuantityDecimal(), precision, orderconstant.DefaultCashPrecision)
	case orderconstant.W_TRX_DIR_SELL:
		orderApplyQuantities, err := wealth.TWealthOrders(orders).ApplyQuantities()
		if err != nil {
			l.Info("collect orders applyQuantities failed", zap.Error(err))
			return code.Response(ctx, code.NewDefaultError(code.ErrUnknow), nil)
		}

		result = quantitycalcuator.AllocateGetAmountByQuantity(orderApplyQuantities,
			r.Params.Ops.FilledAmountDecimal(), r.Params.Ops.FilledQuantityDecimal(), precision, orderconstant.DefaultCashPrecision)
	default:
		l.Info("order direction unknow", zap.Int("nomineeOrderId", r.Params.OrderId), zap.Int("direction", m.Direction))
		return code.Response(ctx, code.NewDefaultError(code.ErrUnknow), nil)
	}

	orderFilledAmounts := result["FinalAmount"]
	orderFilledQuantity := result["FinalQuantity"]

	for i, order := range orders {
		filledAmount := orderFilledAmounts[i].StringFixed(orderconstant.DefaultCashPrecision)
		filledQuantity := orderFilledQuantity[i].RoundDown(precision).StringFixed(precision)
		resp.Orders = append(resp.Orders, mapping.NomineeOrderConfirmPreCheckResultOrder{
			OrderId:        order.Id,
			OrderNumber:    order.OrderNumber,
			SubAccountId:   order.SubAccountId,
			ApplyAmount:    order.ApplyAmount,
			ApplyQuantity:  order.ApplyQuantity,
			ApplyPrice:     order.ApplyPrice,
			FilledAmount:   filledAmount,
			FilledQuantity: filledQuantity,
			FilledPrice:    r.Params.Ops.FilledPrice,
			SourceType:     order.SourceType,
		})
	}

	if err := resp.Fix(precision); err != nil {
		l.Info("fix resp data failed", zap.Error(err))
		return code.Response(ctx, code.NewDefaultError(code.ErrBuildResp), nil)
	}

	return code.Response(ctx, nil, resp)
}
```

这段代码是用Go语言编写的，属于一个名为`wealthAdminService`的结构体的方法实现。该方法`NomineeOrderConfirmPreCheck`主要用于执行提名订单确认的前置检查。以下是代码的主要功能和逻辑流程：

1. **日志和上下文处理**：
   - 从上下文中获取日志实例和用户上下文。
   - 初始化返回的响应结构体`NomineeOrderConfirmPreCheckResult`。

2. **数据库查询**：
   - 通过订单ID查询提名订单。如果订单不存在或查询失败，会记录日志并返回相应的错误响应。

3. **订单状态验证**：
   - 检查订单的当前状态是否允许进行状态变更，如果不符合预期的状态（例如必须是“上手已受理”状态），则返回错误。

4. **数据模型复制**：
   - 使用`copier`库将数据库模型数据复制到响应模型中。如果复制过程中发生错误，会记录日志并返回错误响应。

5. **订单精度和产品信息获取**：
   - 根据产品代码获取产品的基本信息，特别是产品的小数位精度。

6. **订单处理和计算**：
   - 根据提名ID获取相关的所有订单，并按申请金额降序和提交时间升序排序。
   - 根据订单的方向（购买或赎回），使用不同的计算逻辑来处理订单份额或金额的分配计算。

7. **订单结果处理**：
   - 遍历计算结果，将计算得到的填充金额和填充份额等信息添加到响应结构体中。

8. **响应数据调整**：
   - 调整响应数据以适应前端或调用方的需要。

9. **返回结果**：
   - 最终构建完成的响应结果会通过`code.Response`方法返回给调用方。

代码中使用了多个外部库和自定义方法，如`log`, `gorm`, `copier`, `code`, `utils`, `decimal`等，这些都是在Go项目中常用的库或工具。整体上，这个函数的逻辑相对复杂，涉及数据库操作、状态检查、数据复制和调整、精度处理等多个方面。