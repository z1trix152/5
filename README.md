```lua
local ffi = require("ffi")
local imgui = require("mimgui")
local faicons = require("fAwesome6")
local inicfg = require("inicfg")

local mimgui = {} do
	mimgui.font_t = (function()
		local this = {}
		local __mode

		local function constructor(base, size, config, ranges, text)
			local builder = imgui.ImFontGlyphRangesBuilder()
			local range = imgui.ImVector_ImWchar()
			
			for i, r in ipairs(ranges) do
				builder:AddRanges(r)
			end
			if type(text) == "string" then
				builder:AddText(text)
			elseif type(text) == "table" then
				builder:AddText(table.concat(text, ""))
			end
			builder:BuildRanges(range)

			local fonts = imgui.GetIO().Fonts
			return fonts[__mode](fonts, base, size, config, range[0].Data)
		end

		local function determinant(_, mode)
			__mode = mode
			return constructor
		end

		return setmetatable(this, { __index = determinant })
	end)()

	mimgui.frame_t = (function()
		local this = {
			frames = {}
		}

		local metatable = {
			__index = {}
		}

		local WM_KEYDOWN = 0x0100
		local WM_KEYUP = 0x0101
		local VK_ESCAPE = 0x1B

		function metatable.__index:condition()
			return self.status and self.status[0]
		end

		function metatable.__index:draw()
			return self._draw
		end

		function metatable.__index:isActive()
			return self._render
		end

		function metatable.__index:unsubscribe()
			return self:Unsubscribe()
		end

		function metatable.__index:switch(status)
			if type(status) == "boolean" then
				self.status[0] = status
			else
				self.status[0] = not self.status[0]
			end
			return self.status[0]
		end

		function metatable:__newindex(key, value)
			if key == "draw" then
				if type(value) ~= "function" then
					return
				end
				rawset(self, "Condition", function() return self:condition() end)
				rawset(self, "_before", self.before)
				rawset(self, "_draw", value)
			else
				rawset(self, key, value)
			end
		end

		function this:insert(data)
			local frame = imgui.OnFrame(function() return false end, function() end)

			data.status = data.status or imgui.new.bool(false)
			data.weight = data.weight or 1

			for key, value in pairs(data) do
				if not frame[key] then
					frame[key] = value
				end
			end

			setmetatable(frame, metatable)
			table.insert(self.frames, frame)
			return frame
		end

		function this:close(status)
			local weight = -1
			for index, frame in ipairs(self.frames) do
				if frame.status[0] and frame.weight ~= 0 and frame.weight > weight then
					weight = frame.weight
				end
			end
			if weight ~= -1 and status then
				return true, 0
			end
			for index, frame in ipairs(self.frames) do
				if frame.status[0] and frame.weight == weight then
					frame.status[0] = false
				end
			end
			return weight ~= -1, 1
		end

		addEventHandler("onWindowMessage", function(message, wparameters, lparameters)
			local down = message == WM_KEYDOWN
			if down or message == WM_KEYUP then
				if wparameters == VK_ESCAPE and this:close(down) then
					consumeWindowMessage(true, false)
				end
			end
		end)

		return setmetatable(this, { __call = this.insert })
	end)()

	mimgui.color_t = (function()
		local this = {}

		ffi.cdef [[
			typedef union color_t {
				struct { uint8_t a, b, g, r; } n_byte;
				uint32_t hex;
			} color_t;
		]]

		ffi.metatype("color_t", { __index = this })

		local function constructor(_, color)
			local t = ffi.new("color_t")
			t.hex = color
			return t
		end

		function this:represent_as_percent()
			return self.n_byte.r / 0xFF, self.n_byte.g / 0xFF, self.n_byte.b / 0xFF, self.n_byte.a / 0xFF
		end

		return setmetatable(this, { __call = constructor })
	end)()

	mimgui.style_t = (function()
		local this = {}

		local base = {
			static = {
				{ key = imgui.Col.ChildBg, value = { 0.12, -0.50 } },
				{ key = imgui.Col.FrameBg, value = { 0.20, 0.20 } },
				{ key = imgui.Col.FrameBgHovered, value = { 0.30, 0.20 } },
				{ key = imgui.Col.FrameBgActive, value = { 0.45, 0.30 } },
				{ key = imgui.Col.WindowBg, value = { 0.00, 0.00 } },
				{ key = imgui.Col.PopupBg, value = { 0.08, 0.94 } },
				{ key = imgui.Col.Border, value = { 0.43, 0.50 } },
				{ key = imgui.Col.BorderShadow, value = { 0.00, 0.00 } },
				{ key = imgui.Col.TitleBg, value = { 0.04, 1.00 } },
				{ key = imgui.Col.TitleBgActive, value = { 0.16, 1.00 } },
				{ key = imgui.Col.TitleBgCollapsed, value = { 0.00, 0.60 } },
				{ key = imgui.Col.MenuBarBg, value = { 0.14, 1.00 } },
				{ key = imgui.Col.ScrollbarBg, value = { 0.02, 0.53 } },
				{ key = imgui.Col.ScrollbarGrab, value = { 0.31, 1.00 } },
				{ key = imgui.Col.ScrollbarGrabHovered, value = { 0.41, 1.00 } },
				{ key = imgui.Col.ScrollbarGrabActive, value = { 0.51, 1.00 } },
				{ key = imgui.Col.ResizeGrip, value = { 0.13, 0.40 } },
				{ key = imgui.Col.ResizeGripHovered, value = { 0.13, 0.60 } },
				{ key = imgui.Col.ResizeGripActive, value = { 0.13, 0.80 } },
				{ key = imgui.Col.PlotLines, value = { 0.61, 1.00 } },
				{ key = imgui.Col.PlotLinesHovered, value = { 1.00, 1.00 } },
				{ key = imgui.Col.PlotHistogram, value = { 0.90, 1.00 } },
				{ key = imgui.Col.PlotHistogramHovered, value = { 1.00, 1.00 } },
				{ key = imgui.Col.TextSelectedBg, value = { 0.26, 0.35 } }
			},
			colored = {
				{ key = imgui.Col.Button, value = 0.00 },
				{ key = imgui.Col.ButtonHovered, value = 0.10 },
				{ key = imgui.Col.ButtonActive, value = 0.15 },
				{ key = imgui.Col.Header, value = 0.00 },
				{ key = imgui.Col.HeaderHovered, value = 0.10 },
				{ key = imgui.Col.HeaderActive, value = 0.15 },
				{ key = imgui.Col.Separator, value = 0.00 },
				{ key = imgui.Col.SeparatorHovered, value = 0.10 },
				{ key = imgui.Col.SeparatorActive, value = 0.15 },
				{ key = imgui.Col.SliderGrab, value = 0.00 },
				{ key = imgui.Col.SliderGrabActive, value = 0.15 },
				{ key = imgui.Col.CheckMark, value = 0.00 }
			}
		}

		local function constructor(_, elements, infobar, text)
			return setmetatable({ elements = elements, infobar = infobar, text = text }, { __index = this })
		end

		function this:apply()
			imgui.SwitchContext()
			local style = imgui.GetStyle()
			local colors = style.Colors

			style.WindowBorderSize       = 0.0
			style.WindowRounding         = 10.0
			style.PopupBorderSize        = 0.0
			style.PopupRounding          = 10.0
			style.ChildRounding          = 10.0
			style.WindowTitleAlign       = imgui.ImVec2(0.5, 0.5)
			style.FrameRounding          = 7.0
			style.ItemSpacing            = imgui.ImVec2(10, 5)
			style.ScrollbarSize          = 10.0
			style.ScrollbarRounding      = 2.0
			style.GrabMinSize            = 5.0
			style.GrabRounding           = 10.0
			style.WindowPadding          = imgui.ImVec2(10, 10)
			style.FramePadding           = imgui.ImVec2(5, 0)
			style.DisplayWindowPadding   = imgui.ImVec2(27, 27)
			style.DisplaySafeAreaPadding = imgui.ImVec2(5, 5)
			style.ButtonTextAlign        = imgui.ImVec2(0.5, 0.5)
			style.IndentSpacing          = 10.0
			style.Alpha                  = 1.0

			local colored = { self.elements:represent_as_percent() }
			local static = { self.infobar:represent_as_percent() }
			local text = { self.text:represent_as_percent() }

			for i, v in ipairs(base.colored) do
				colors[v.key] = imgui.ImVec4(colored[1] + v.value, colored[2] + v.value, colored[3] + v.value, 1 - v.value)
			end

			for i, v in ipairs(base.static) do
				colors[v.key] = imgui.ImVec4(
					static[1] + v.value[1] * (static[1] > 0.5 and -1 or 1),
					static[2] + v.value[1] * (static[2] > 0.5 and -1 or 1),
					static[3] + v.value[1] * (static[3] > 0.5 and -1 or 1),
					static[4] + v.value[2]
				)
			end

			colors[imgui.Col.Text] = imgui.ImVec4(text[1], text[2], text[3], 1)
			colors[imgui.Col.TextDisabled] = imgui.ImVec4(
				text[1] + (text[1] > 0.5 and -0.3 or 0.3),
				text[2] + (text[2] > 0.5 and -0.3 or 0.3),
				text[3] + (text[3] > 0.5 and -0.3 or 0.3),
				text[4]
			)
		end

		return setmetatable(this, { __call = constructor })
	end)()

	mimgui.animation_t = (function()
		local this = {}
		local metatable = {}
		local ui = {}

		local function bringVec4To(from, to, sclock, duration)
			local clock = os.clock() - sclock
			if clock >= 0 and clock <= duration then
				local k = clock / (duration / 100)
				return imgui.ImVec4 (
					from.x + (k * (to.x - from.x) / 100),
					from.y + (k * (to.y - from.y) / 100),
					from.z + (k * (to.z - from.z) / 100),
					from.w + (k * (to.w - from.w) / 100)
				), true
			end
			return (clock > duration) and to or from, false
		end

		function this:init(element, default, hovered, active, duration)
			local t = {}
			local m = {}

			t.__element = element
			t.__duration = duration or 0.4
			t.__colors = {
				default = default,
				hovered = hovered,
				active = active
			}

			function m:__call(str_id, ...)
				local result, color, status, index

				if self[str_id].clock then
					color, status = bringVec4To(self[str_id].from, self[str_id].to, self[str_id].clock, self.__duration)
					if not status then
						self[str_id].clock = false
					end
				else
					color = imgui.GetStyle().Colors[self[str_id].index]
				end

				imgui.PushStyleColor(self.__colors.default, color)
				imgui.PushStyleColor(self.__colors.hovered, color)
				imgui.PushStyleColor(self.__colors.active, color)
				result = imgui[self.__element](str_id, ...)
				imgui.PopStyleColor(3)

				if imgui.IsItemActive() then
					index = self.__colors.active
				elseif imgui.IsItemHovered() then 
					index = self.__colors.hovered
				else
					index = self.__colors.default
				end

				if self[str_id].index ~= index then
					self[str_id] = {
						index = index,
						clock = os.clock(),
						from = imgui.GetStyle().Colors[self[str_id].index],
						to = imgui.GetStyle().Colors[index]
					}
				end

				return result
			end

			function m:__index(k)
				self[k] = { index = self.__colors.default }
				return self[k]
			end

			ui[element] = setmetatable(t, m)
		end

		function metatable:__index(element)
			return ui[element]
		end

		metatable.__call = this.init

		return setmetatable(this, metatable)
	end)()

	mimgui.animation_scroll = (function()
		local this = {}
		local flags = imgui.HoveredFlags.AnyWindow

		local ui = setmetatable({}, {
			__index = function(t, k)
				t[k] = {}
				return t[k]
			end
		})

		function this:call(str_id, step, duration, flag)
			local scroll = imgui.GetScrollY()
			local el = ui[str_id]
			flag = flag or flags
		
			if imgui.IsWindowHovered(flag) and imgui.IsMouseDown(0) then
				el.sclock = false
			end

			if el.sclock then
				if os.clock() - el.sclock <= duration then
					local k = (os.clock() - el.sclock) / duration
					local fk = k * (2 - k)
					local n = el.sposition + (el.tposition - el.sposition) * (k * (2 - k))

					if n < 0 then
						n = 0
						el.sclock = false
					elseif n > imgui.GetScrollMaxY() then
						n = imgui.GetScrollMaxY()
						el.sclock = false
					end

					imgui.SetScrollY(math.floor(n))
				else
					el.sclock = false
					imgui.SetScrollY(el.tposition)
				end
			end

			local wheel = imgui.GetIO().MouseWheel

			if wheel ~= 0 and imgui.IsWindowHovered(flag) then
				local offset = -wheel * step

				if el.sclock then
					el.sclock = os.clock()
					el.sposition = scroll

					if el.sposition < el.tposition and offset > 0 then
						el.tposition = el.tposition + offset
					elseif el.sposition > el.tposition and offset < 0 then
						el.tposition = el.tposition + offset
					else
						el.tposition = scroll + offset
					end
				else
					el.sclock = os.clock()
					el.sposition = scroll
					el.tposition = scroll + offset
				end
			end

			return wheel ~= 0, imgui.GetScrollY(), imgui.GetScrollMaxY()
		end

		return setmetatable(this, { __call = this.call })
	end)()

	mimgui.hint = (function()
		local pool = {}
		return function(str_id, text)
			if imgui.IsItemHovered() then
				if not pool[str_id] then
					pool[str_id] = os.clock()
				end
				if os.clock() - pool[str_id] > 0.3 then
					imgui.SetTooltip(text)
				end
			elseif pool[str_id] then
				pool[str_id] = nil
			end
		end
	end)()
end

imgui.OnInitialize(function()
	local fontPath = getFolderPath(0x14) .. "\\trebucbd.ttf"
	fontSize = {}
	imgui.GetIO().Fonts:Clear()

	local config = imgui.ImFontConfig()
	config.MergeMode = true
	config.PixelSnapH = true

	fontSize.b15 = mimgui.font_t.AddFontFromFileTTF(fontPath, 15, nil, { imgui.GetIO().Fonts:GetGlyphRangesCyrillic() }, "љњЋЌЎЏђѓѕѕјљњћќўџҐҐЁёєѕіІї")
	fontSize.i15 = mimgui.font_t.AddFontFromMemoryCompressedBase85TTF(faicons.get_font_data_base85("solid"), 15, config, {}, {
		faicons("xmark"),
		faicons("gear"),
		faicons("ban")
	})

	fontSize.b18 = mimgui.font_t.AddFontFromFileTTF(fontPath, 18, nil, { imgui.GetIO().Fonts:GetGlyphRangesCyrillic() }, "љњЋЌЎЏђѓѕѕјљњћќўџҐҐЁёєѕіІї")

	fontSize.i32 = mimgui.font_t.AddFontFromMemoryCompressedBase85TTF(faicons.get_font_data_base85("solid"), 32, config, {}, {
		faicons("crosshairs")
	})

	mimgui.animation_t("Button", imgui.Col.Button, imgui.Col.ButtonHovered, imgui.Col.ButtonActive, 0.4)
	mimgui.animation_t("NoAlphaButton", imgui.Col.Text, imgui.Col.TextDisabled, imgui.Col.Text, 0.4)
	mimgui.animation_t("RadioButtonIntPtr", imgui.Col.FrameBg, imgui.Col.FrameBgHovered, imgui.Col.FrameBgActive, 0.4)
	mimgui.animation_t("SliderFloat", imgui.Col.FrameBg, imgui.Col.FrameBgHovered, imgui.Col.FrameBgActive, 0.4)
	mimgui.animation_t("Checkbox", imgui.Col.FrameBg, imgui.Col.FrameBgHovered, imgui.Col.FrameBgActive, 0.4)

	mimgui.style_t(mimgui.color_t(0x8A95A4FF), mimgui.color_t(0x121212E0), mimgui.color_t(0xFFFFFFFF)):apply()
end)

local window = (function()
	local this = mimgui.frame_t {
		flags = imgui.WindowFlags.NoCollapse + imgui.WindowFlags.NoTitleBar + imgui.WindowFlags.AlwaysAutoResize
	}

	local mode = ffi.new("int[1]", 0)
	local float = ffi.new("float[5][1]")
	local bool = ffi.new("bool[21][1]")

	local cfg = inicfg.load {
		smooth = {},
		silent = {},
		nospread = {}
	}

	local data = setmetatable({}, {
		__index = function(t, b)
			t[b] = setmetatable({}, {
				__index = function(t, n)
					t[n] = setmetatable({}, {
						__index = function(t, k)
							if not cfg[b][k] then
								if n == "bool" then
									cfg[b][k] = false
								else
									cfg[b][k] = 0
								end
							end
							t[k] = ffi.new(n .. "[1]", cfg[b][k])
							return t[k]
						end
					})
					return t[n]
				end
			})
			return t[b]
		end
	})

	ffi.cdef [[
		typedef struct Bone {
			ImVec2 pos;
			bool status[1];
		} Bone;
	]]

	local bones = ffi.new("Bone[6]") do
		bones[0].pos.x, bones[0].pos.y = 67, 10
		bones[1].pos.x, bones[1].pos.y = 67, 60

		bones[2].pos.x, bones[2].pos.y = 25, 55
		bones[3].pos.x, bones[3].pos.y = 110, 55

		bones[4].pos.x, bones[4].pos.y = 47, 120
		bones[5].pos.x, bones[5].pos.y = 87, 120
	end

	local function draw_bool(size, str_id, name, set, key)
		imgui.SetNextWindowBgAlpha(0.75)
		imgui.BeginChild(str_id .. name, imgui.ImVec2(size, 25))
			imgui.SetCursorPos(imgui.ImVec2(5, 5))
			if mimgui.animation_t.Checkbox(name, data[set].bool[key]) then
				cfg[set][key] = data[set].bool[key][0]
				inicfg.save(cfg)
			end
		imgui.EndChild()
	end

	local function draw_float(size, str_id, name, set, key, format)
		imgui.SetNextWindowBgAlpha(0.75)
		imgui.BeginChild(str_id .. name, imgui.ImVec2(size, 25))
			imgui.SetCursorPos(imgui.ImVec2(5, 5))
			imgui.PushItemWidth(125)
			if mimgui.animation_t.SliderFloat(name, data[set].float[key], 0, 100, format) then
				cfg[set][key] = data[set].float[key][0]
				inicfg.save(cfg)
			end
		imgui.EndChild()
	end

	function this:draw()
		imgui.Begin("##window", self.status, self.flags)
			imgui.SetCursorPos(imgui.ImVec2(15, 25))
			imgui.PushFont(fontSize.i32)
			imgui.Text(faicons("crosshairs"))
			imgui.PopFont()

			imgui.SetCursorPos(imgui.ImVec2(55, 10))
			imgui.BeginGroup()
				imgui.PushStyleVarVec2(imgui.StyleVar.ItemSpacing, imgui.ImVec2(0, 0))
				imgui.PushFont(fontSize.b18)
				imgui.Text("AimHelper SRP")
				imgui.PopFont()
				imgui.PopStyleVar(1)
				imgui.TextDisabled("build 1.0")
			imgui.EndGroup()

			local size = imgui.GetWindowSize()
			imgui.SameLine(size.x - 30)
			if mimgui.animation_t.NoAlphaButton(faicons("xmark"), imgui.ImVec2(18, 18)) then
				self:switch()
			end

			imgui.Spacing()

			imgui.BeginGroup()
				imgui.BeginChild("##mode", imgui.ImVec2(150, 105))
					imgui.SetCursorPos(imgui.ImVec2(10, 10))
					imgui.BeginGroup()
						imgui.SetNextWindowBgAlpha(0.75)
						imgui.BeginChild("##mode__smooth", imgui.ImVec2(130, 25))
							imgui.SetCursorPos(imgui.ImVec2(5, 5))
							mimgui.animation_t.RadioButtonIntPtr("Smooth", mode, 0)
						imgui.EndChild()

						imgui.SetNextWindowBgAlpha(0.75)
						imgui.BeginChild("##mode__silent", imgui.ImVec2(130, 25))
							imgui.SetCursorPos(imgui.ImVec2(5, 5))
							mimgui.animation_t.RadioButtonIntPtr("Silent", mode, 1)
						imgui.EndChild()


						imgui.SetNextWindowBgAlpha(0.75)
						imgui.BeginChild("##mode__nospread", imgui.ImVec2(130, 25))
							imgui.SetCursorPos(imgui.ImVec2(5, 5))
							mimgui.animation_t.RadioButtonIntPtr("NoSpread", mode, 2)
						imgui.EndChild()
					imgui.EndGroup()
				imgui.EndChild()

				if mode[0] < 2 then
					imgui.Spacing()
					imgui.BeginChild("##bones", imgui.ImVec2(150, 185 + (mode[0] == 0 and 10 or 0)))
						local p = imgui.GetCursorScreenPos()
						local dl = imgui.GetWindowDrawList()
						-- Рисуем фон-заглушку вместо текстуры скина
						dl:AddRectFilled(p, imgui.ImVec2(p.x + 140, p.y + 175), 0x22FFFFFF, 6.0)

						for i = 0, 5 do
							imgui.SetCursorPos(bones[i].pos)
							mimgui.animation_t.Checkbox("##" .. i, bones[i].status)
						end
					imgui.EndChild()
				end
			imgui.EndGroup()

			imgui.SameLine()

			if mode[0] == 0 then
				imgui.SetNextWindowBgAlpha(0)
				imgui.BeginChild("##settings__smooth", imgui.ImVec2(250, 310))
					imgui.BeginChild("##settings__smooth__base", imgui.ImVec2(250, 310))
						imgui.SetCursorPos(imgui.ImVec2(10, 10))
						imgui.Text(faicons("gear") .. " Base settings")
						imgui.Spacing()

						imgui.SetCursorPosX(10)
						imgui.SetNextWindowBgAlpha(0)
						imgui.BeginChild("##settings__smooth__base__body", imgui.ImVec2(230, 270), nil, imgui.WindowFlags.NoScrollWithMouse)
							mimgui.animation_scroll("##settings__smooth__base__body__scroll", 50, 0.4)

							local str_id = "##settings__smooth__base"
							draw_float(230, str_id, "FOV", "smooth", "fov", "%.1f°")
							draw_float(230, str_id, "Smooth", "smooth", "smooth", "%.1f")
							
							draw_bool(230, str_id, "Enabled", "smooth", "status")
							draw_bool(230, str_id, "Dynamic FOV", "smooth", "dfov")
							draw_bool(230, str_id, "Aim on fire", "smooth", "aimonfire")
							draw_bool(230, str_id, "Aim on Y axis", "smooth", "aimony")
							draw_bool(230, str_id, "Nearest bone", "smooth", "nearest")
							draw_bool(230, str_id, "Model filter", "smooth", "mfilter")
							draw_bool(230, str_id, "Clist filter", "smooth", "cfilter")
							imgui.Spacing()
						imgui.EndChild()
					imgui.EndChild()
				imgui.EndChild()
			elseif mode[0] == 1 then
				imgui.SetNextWindowBgAlpha(0)
				imgui.BeginChild("##settings__silent", imgui.ImVec2(250, 300))
					imgui.BeginChild("##settings__silent__base", imgui.ImVec2(250, 160))
						imgui.SetCursorPos(imgui.ImVec2(10, 10))
						imgui.Text(faicons("gear") .. " Base settings")
						imgui.Spacing()

						imgui.SetCursorPosX(10)
						imgui.SetNextWindowBgAlpha(0)
						imgui.BeginChild("##settings__silent__base__body", imgui.ImVec2(230, 120), nil, imgui.WindowFlags.NoScrollWithMouse)
							mimgui.animation_scroll("##settings__silent_base__body__scroll", 50, 0.4)

							local str_id = "##settings__silent__base"
							draw_float(215, str_id, "FOV", "silent", "fov", "%.1f°")
							draw_float(215, str_id, "Chanse", "silent", "chanse", "%.0f%%")

							draw_bool(215, str_id, "Enabled", "silent", "status")
							draw_bool(215, str_id, "WallShot", "silent", "wallshot")
							draw_bool(215, str_id, "Dynamic FOV", "silent", "dfov")
							draw_bool(215, str_id, "Set rotation", "silent", "srot")
							draw_bool(215, str_id, "Set target", "silent", "star")
							draw_bool(215, str_id, "Model filter", "silent", "mfilter")
							draw_bool(215, str_id, "Clist filter", "silent", "cfilter")
							imgui.Spacing()
						imgui.EndChild()
					imgui.EndChild()

					imgui.Spacing()

					imgui.BeginChild("##settings__silent__collision", imgui.ImVec2(250, 130))
						imgui.SetCursorPos(imgui.ImVec2(10, 10))
						imgui.BeginGroup()
							imgui.Text(faicons("ban") .. " Collision")
							imgui.Spacing()

							draw_bool(230, str_id, "Vehicle", "silent", "cveh")
							draw_bool(230, str_id, "Building", "silent", "cbld")
							draw_bool(230, str_id, "Objects", "silent", "cobj")
						imgui.EndGroup()
					imgui.EndChild()
				imgui.EndChild()
			elseif mode[0] == 2 then
				imgui.BeginChild("##settings__nospread", imgui.ImVec2(250, 100))
					imgui.SetCursorPos(imgui.ImVec2(10, 10))
					imgui.BeginGroup()
						imgui.Text(faicons("gear") .. " Base settings")
						imgui.Spacing()

						local str_id = "##settings__nospread"
						draw_bool(230, str_id, "Enabled", "nospread", "status")
						draw_float(230, str_id, "Percent", "nospread", "percent", "%.0f%%")
					imgui.EndGroup()
				imgui.EndChild()
			end
		imgui.End()
	end

	return this
end)()

function main()
	while not isSampAvailable() do wait(0) end

	sampRegisterChatCommand("asrp", function()
		window:switch()
	end)

	addEventHandler("onWindowMessage", function(message, wparameters, lparameters)
		if message == 0x101 then
			if wparameters == 0x2D then
				window:switch()
			end
		end
	end)

	wait(-1)
end

function imgui.NoAlphaButton(str_id, size)
	local color = imgui.ImVec4(0, 0, 0, 0)
  imgui.PushStyleColor(imgui.Col.Button, color)
  imgui.PushStyleColor(imgui.Col.ButtonActive, color)
  imgui.PushStyleColor(imgui.Col.ButtonHovered, color)
  local result = imgui.Button(str_id, size)
  imgui.PopStyleColor(3)
  return result
end
```
