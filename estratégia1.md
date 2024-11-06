    def Initialize(self):
        # Configuración básica
        self.SetStartDate(2022, 1, 1)  # Fecha de inicio del backtest, hace 1 año
        self.SetEndDate(2023, 1, 1)  # Fecha de fin del backtest
        self.SetCash(100000)  # Capital inicial de $100,000

        # Configuración del activo - CFD del índice DAX
        self.dax = self.AddCfd("DE30EUR", Resolution.Minute, Market.Oanda).Symbol

        # Definición de parámetros para optimización
        self.take_profit = int(self.GetParameter("take_profit", 9))  # Normal 9, min 4, max 20
        self.exit_point = int(self.GetParameter("exit_point", 6))    # Normal 6, min 4, max 20
        self.entry_minute = int(self.GetParameter("entry_minute", 5)) # Normal 5, min 1, max 60

        # Variables de la estrategia
        self.high = None
        self.low = None
        self.observation_period = False
        self.active_trade = None  # Para verificar si hay una operación activa

        # Tickets de órdenes para gestionar órdenes activas
        self.buy_order_ticket = None
        self.sell_order_ticket = None

    def OnData(self, data):
        # Configurar el horario de observación de 7:00 a entry_minute
        if self.Time.hour == 7 and self.Time.minute == 0:
            # Iniciar el período de observación
            self.high = None
            self.low = None
            self.observation_period = True
            self.active_trade = None  # Reiniciar cualquier operación activa

        if self.observation_period and self.Time.hour == 7 and self.Time.minute < self.entry_minute:
            # Actualizar el alto y el bajo durante el período de observación
            if data.ContainsKey(self.dax):
                price = data[self.dax].Price
                self.high = max(self.high, price) if self.high else price
                self.low = min(self.low, price) if self.low else price

        if self.Time.hour == 7 and self.Time.minute == self.entry_minute:
            # Finalizar el período de observación
            self.observation_period = False
            self.PlaceOrders()
    
    def PlaceOrders(self):
        # Colocar órdenes de compra y venta según el alto y bajo observados
        if self.high and self.low:
            # Calcular tamaño de la posición basándonos en el riesgo de 0.3% del capital
            capital_risk = self.Portfolio.Cash * 0.003
            quantity = int(capital_risk / self.exit_point)  # exit_point como el riesgo en puntos

            # Crear órdenes con stop loss y take profit
            self.buy_order_ticket = self.StopMarketOrder(self.dax, quantity, self.high)
            self.sell_order_ticket = self.StopMarketOrder(self.dax, -quantity, self.low)
            self.Log(f"Buy Order at {self.high}, Sell Short Order at {self.low}")

    def OnOrderEvent(self, orderEvent):
        # Verificar si una de las órdenes ha sido llenada
        if orderEvent.Status == OrderStatus.Filled:
            # Si se ejecuta una orden, cancelar la opuesta
            if self.buy_order_ticket is not None and orderEvent.OrderId == self.buy_order_ticket.OrderId:
                self.active_trade = "long"
                if self.sell_order_ticket is not None:
                    self.sell_order_ticket.Cancel()
            elif self.sell_order_ticket is not None and orderEvent.OrderId == self.sell_order_ticket.OrderId:
                self.active_trade = "short"
                if self.buy_order_ticket is not None:
                    self.buy_order_ticket.Cancel()

            # Configurar stop loss y take profit utilizando parámetros
            entry_price = orderEvent.FillPrice
            if self.active_trade == "long":
                stop_price = entry_price - self.exit_point  # exit_point puntos abajo
                take_profit_price = entry_price + self.take_profit  # take_profit puntos arriba
                self.StopMarketOrder(self.dax, -self.buy_order_ticket.Quantity, stop_price)
                self.LimitOrder(self.dax, -self.buy_order_ticket.Quantity, take_profit_price)
            elif self.active_trade == "short":
                stop_price = entry_price + self.exit_point  # exit_point puntos arriba
                take_profit_price = entry_price - self.take_profit  # take_profit puntos abajo
                self.StopMarketOrder(self.dax, -self.sell_order_ticket.Quantity, stop_price)
                self.LimitOrder(self.dax, -self.sell_order_ticket.Quantity, take_profit_price)