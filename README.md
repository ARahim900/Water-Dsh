import React, { useState, useEffect } from 'react';
import { 
  BarChart, Bar, LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer, 
  PieChart, Pie, Cell, Area, AreaChart, ComposedChart
} from 'recharts';
import Papa from 'papaparse';
import { Droplet, Home, ChevronRight, ArrowUpDown, Search, ChevronDown, TrendingUp, TrendingDown, BarChart2, Calendar, Filter, AlertCircle, FileText, Activity } from 'lucide-react';

// Enhanced color scheme with gauges matching screenshot - loss gauges in red
const COLORS = {
  primary: '#4F7ED9',  // Blue for L2 Bulk
  secondary: '#50C7B0', // Teal for L3 Consumption
  tertiary: '#EF476F',  // Red for other uses
  lossVolume: '#EF476F', // Red for Loss Volume
  lossPercentage: '#EF476F', // Red for Loss Percentage
  danger: '#EF476F',
  warning: '#F9C74F',
  background: '#F8F9FC',
  cardBg: '#FFFFFF',
  header: '#4E4456',
  text: {
    primary: '#3E4B5C',
    secondary: '#6B7A90',
    light: '#9AA5B6'
  },
  border: '#E5E9F2',
  chart: ['#4F7ED9', '#50C7B0', '#6EC561', '#9E84E3', '#EF476F', '#F9C74F', '#FB5607', '#4CC9F0']
};

// Circular gauge component styled to match screenshot exactly
const CircularGauge = ({ value, total, color, label, unit = 'm³', overrideValue = null, isPercentage = false }) => {
  const displayValue = overrideValue !== null ? overrideValue : value;
  const percentage = Math.min(Math.max((value / total) * 100, 0), 100);
  const radius = 70;
  const strokeWidth = 12;
  const circumference = 2 * Math.PI * radius;
  const strokeDashoffset = circumference - (percentage / 100) * circumference;
  
  return (
    <div className="flex flex-col items-center">
      <div className="relative mb-4">
        <svg width="180" height="180" viewBox="0 0 180 180" className="transform -rotate-90">
          {/* Background circle */}
          <circle 
            cx="90" 
            cy="90" 
            r={radius} 
            fill="none" 
            stroke="#F0F2F8" 
            strokeWidth={strokeWidth} 
          />
          
          {/* Foreground circle without gradient to match screenshot */}
          <circle 
            cx="90" 
            cy="90" 
            r={radius} 
            fill="none" 
            stroke={color} 
            strokeWidth={strokeWidth} 
            strokeDasharray={circumference} 
            strokeDashoffset={strokeDashoffset} 
            strokeLinecap="round" 
          />
        </svg>
        <div className="absolute top-0 left-0 w-full h-full flex flex-col items-center justify-center">
          <span className="text-3xl font-bold text-gray-800">
            {isPercentage ? 
              displayValue.toFixed(2) : 
              displayValue.toLocaleString()}
          </span>
          {isPercentage && (
            <span className="text-sm text-gray-500">100%</span>
          )}
          {!isPercentage && (
            <span className="text-sm text-gray-500">{unit}</span>
          )}
        </div>
      </div>
      <div className="text-center">
        <p className="text-sm font-medium text-gray-600">{label}</p>
      </div>
    </div>
  );
};

// Enhanced horizontal bar chart for consumption in table
const ConsumptionBar = ({ value, maxValue }) => {
  const percentage = Math.min((value / maxValue) * 100, 100);
  
  return (
    <div className="flex items-center">
      <div className="w-32 h-2 bg-gray-200 rounded-full mr-3">
        <div 
          className="h-full rounded-full" 
          style={{ 
            width: `${percentage}%`,
            background: `linear-gradient(90deg, ${COLORS.primary}80 0%, ${COLORS.primary} 100%)`
          }}
        />
      </div>
      <span className="text-sm text-gray-800 font-medium">{value} m³</span>
    </div>
  );
};

// Enhanced dropdown selector component with animation
const Dropdown = ({ value, options, onChange, className }) => {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div className={`relative inline-block ${className}`}>
      <div 
        className="inline-flex items-center justify-between rounded-md border border-gray-300 bg-white px-3 py-1.5 text-sm shadow-sm cursor-pointer transition-all hover:border-gray-400"
        onClick={() => setIsOpen(!isOpen)}
      >
        <span>{options.find(opt => opt.value === value)?.label || value}</span>
        <ChevronDown size={16} className={`ml-2 text-gray-500 transition-transform duration-200 ${isOpen ? 'rotate-180' : ''}`} />
      </div>
      
      {isOpen && (
        <div className="absolute z-10 mt-1 w-full bg-white rounded-md shadow-lg border border-gray-200 py-1 max-h-60 overflow-auto">
          {options.map(option => (
            <div 
              key={option.value} 
              className={`px-3 py-1.5 text-sm cursor-pointer hover:bg-gray-50 transition-colors ${option.value === value ? 'bg-blue-50 text-blue-600' : ''}`}
              onClick={() => {
                onChange(option.value);
                setIsOpen(false);
              }}
            >
              {option.label}
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

// Tab button component
const TabButton = ({ active, onClick, children }) => {
  return (
    <button 
      onClick={onClick}
      className={`py-4 px-6 font-medium text-sm border-b-2 transition-colors duration-200 ${
        active 
          ? 'border-blue-500 text-blue-600' 
          : 'border-transparent text-gray-500 hover:text-gray-700 hover:border-gray-300'
      }`}
    >
      {children}
    </button>
  );
};

// Card component with hover effect
const Card = ({ children, className = '' }) => {
  return (
    <div className={`bg-white rounded-lg shadow-sm border border-gray-200 transition-all duration-200 hover:shadow-md ${className}`}>
      {children}
    </div>
  );
};

// Main dashboard component
const DashboardPage = () => {
  const [masterData, setMasterData] = useState(null);
  const [selectedMonth, setSelectedMonth] = useState('Mar-25');
  const [selectedZone, setSelectedZone] = useState(null);
  const [selectedType, setSelectedType] = useState('all');
  const [metrics, setMetrics] = useState(null);
  const [zoneMetrics, setZoneMetrics] = useState({});
  const [typeBreakdown, setTypeBreakdown] = useState([]);
  const [zoneMeters, setZoneMeters] = useState([]);
  const [searchTerm, setSearchTerm] = useState('');
  const [activeTab, setActiveTab] = useState('zone');
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    async function fetchData() {
      try {
        setIsLoading(true);
        const response = await window.fs.readFile('2025 Water ConsumptionsMaster View Water 2025 9.csv', { encoding: 'utf8' });
        
        const parsedData = Papa.parse(response, {
          header: true,
          dynamicTyping: true,
          skipEmptyLines: true
        });
        
        setMasterData(parsedData.data);
        
        // Set first zone as default selected zone
        const zones = [...new Set(parsedData.data.filter(m => m.Zone).map(m => m.Zone))];
        if (zones.length > 0) {
          setSelectedZone(zones[0]);
        }
        
        // Calculate metrics for the selected month
        calculateMetrics(parsedData.data, selectedMonth, zones[0]);
        
        setIsLoading(false);
      } catch (error) {
        console.error('Error fetching data:', error);
        setIsLoading(false);
      }
    }
    
    fetchData();
  }, []);
  
  useEffect(() => {
    if (masterData && selectedZone) {
      calculateMetrics(masterData, selectedMonth, selectedZone);
    }
  }, [masterData, selectedMonth, selectedZone]);
  
  const calculateMetrics = (data, month, zone) => {
    // Calculate L1, L2, L3 volumes
    const l1Supply = data
      .filter(meter => meter.Label === 'L1')
      .reduce((sum, meter) => sum + (meter[month] || 0), 0);
      
    const l2Volume = data
      .filter(meter => meter.Label === 'L2' || (meter.Label === 'DC' && 
              data.find(m => m.Label === 'L1' && m['Meter Label'] === meter['Parent Meter'])))
      .reduce((sum, meter) => sum + (meter[month] || 0), 0);
      
    const l3Volume = data
      .filter(meter => meter.Label === 'L3' || meter.Label === 'DC')
      .filter(meter => meter['Acct #'] !== '4300322') // Exclude the anomaly meter
      .reduce((sum, meter) => sum + (meter[month] || 0), 0);
      
    // Calculate losses
    const stage1Loss = l1Supply - l2Volume;
    const stage2Loss = l2Volume - l3Volume;
    const totalLoss = l1Supply - l3Volume;
    
    // Calculate percentages
    const stage1LossPercent = (stage1Loss / l1Supply) * 100;
    const stage2LossPercent = (stage2Loss / l2Volume) * 100;
    const totalLossPercent = (totalLoss / l1Supply) * 100;
    
    // Set overall metrics
    setMetrics({
      l1Supply,
      l2Volume,
      l3Volume,
      stage1Loss,
      stage2Loss,
      totalLoss,
      stage1LossPercent,
      stage2LossPercent,
      totalLossPercent
    });
    
    // Calculate zone-specific metrics
    const zones = [...new Set(data.filter(m => m.Zone).map(m => m.Zone))];
    const zoneData = {};
    
    zones.forEach(zoneId => {
      const zoneBulkMeter = data.find(m => m.Label === 'L2' && m.Zone === zoneId);
      if (!zoneBulkMeter) return;
      
      const zoneBulkReading = zoneBulkMeter[month] || 0;
      
      const zoneL3Sum = data
        .filter(m => m.Label === 'L3' && m.Zone === zoneId && m['Acct #'] !== '4300322')
        .reduce((sum, meter) => sum + (meter[month] || 0), 0);
        
      const internalZoneLoss = zoneBulkReading - zoneL3Sum;
      const internalZoneLossPercent = zoneBulkReading > 0 ? (internalZoneLoss / zoneBulkReading) * 100 : 0;
      const isNegativeLoss = internalZoneLoss < 0;
      
      zoneData[zoneId] = {
        zoneName: zoneId,
        bulkReading: zoneBulkReading,
        l3Sum: zoneL3Sum,
        loss: Math.abs(internalZoneLoss),
        isNegativeLoss: isNegativeLoss,
        lossPercent: Math.abs(internalZoneLossPercent)
      };
    });
    
    setZoneMetrics(zoneData);
    
    // Calculate consumption by type
    const types = [...new Set(data.filter(m => m.Type).map(m => m.Type))];
    const typeData = [];
    
    types.forEach(type => {
      const typeConsumption = data
        .filter(m => m.Type === type && (m.Label === 'L3' || m.Label === 'DC') && m['Acct #'] !== '4300322')
        .reduce((sum, meter) => sum + (meter[month] || 0), 0);
        
      if (typeConsumption > 0) {
        typeData.push({
          name: type,
          value: typeConsumption
        });
      }
    });
    
    setTypeBreakdown(typeData);
    
    // Get zone meters
    if (zone) {
      const meters = data
        .filter(meter => meter.Zone === zone && meter.Label === 'L3')
        .map(meter => ({
          accountNumber: meter['Acct #'],
          meterLabel: meter['Meter Label'],
          parentMeter: meter['Parent Meter'],
          type: meter.Type,
          consumption: meter[month] || 0
        }))
        .sort((a, b) => b.consumption - a.consumption);
      
      setZoneMeters(meters);
    }
  };
  
  // Format number with commas and fixed decimal places
  const formatNumber = (num, decimals = 0) => {
    if (num === null || num === undefined) return '-';
    return num.toLocaleString('en-US', { 
      minimumFractionDigits: decimals,
      maximumFractionDigits: decimals
    });
  };
  
  // Filter meters based on search term
  const getFilteredMeters = () => {
    if (!searchTerm) return zoneMeters;
    
    const lowerSearchTerm = searchTerm.toLowerCase();
    return zoneMeters.filter(meter => 
      meter.meterLabel.toLowerCase().includes(lowerSearchTerm) ||
      meter.accountNumber?.toString().toLowerCase().includes(lowerSearchTerm) ||
      meter.type?.toLowerCase().includes(lowerSearchTerm)
    );
  };
  
  // Get max consumption for bar scale
  const getMaxConsumption = () => {
    if (!zoneMeters || zoneMeters.length === 0) return 1000;
    return Math.max(...zoneMeters.map(meter => meter.consumption)) || 1000;
  };
  
  // Get top loss zones for chart
  const getTopLossZones = () => {
    if (!zoneMetrics) return [];
    
    return Object.values(zoneMetrics)
      .filter(zone => zone.bulkReading > 0) // Only include zones with readings
      .sort((a, b) => b.lossPercent - a.lossPercent)
      .slice(0, 5)
      .map(zone => ({
        name: zone.zoneName.replace('Zone_', '').replace('_', ' '),
        lossPercent: parseFloat(zone.lossPercent.toFixed(1)),
        volume: zone.loss
      }));
  };
  
  // Get trend data for charts
  const getTrendData = () => {
    if (!masterData) return [];
    
    const months = ['Jan-25', 'Feb-25', 'Mar-25'];
    return months.map(month => {
      const l1Supply = masterData
        .filter(meter => meter.Label === 'L1')
        .reduce((sum, meter) => sum + (meter[month] || 0), 0);
        
      const l3Volume = masterData
        .filter(meter => meter.Label === 'L3' || meter.Label === 'DC')
        .filter(meter => meter['Acct #'] !== '4300322')
        .reduce((sum, meter) => sum + (meter[month] || 0), 0);
        
      const totalLoss = l1Supply - l3Volume;
      const totalLossPercent = (totalLoss / l1Supply) * 100;
      
      return {
        name: month.replace('-25', ''),
        l1Supply,
        l3Consumption: l3Volume,
        loss: totalLoss,
        lossPercent: parseFloat(totalLossPercent.toFixed(1))
      };
    });
  };
  
  // Get consumption data for type breakdown
  const getTypeBreakdownData = () => {
    return typeBreakdown.filter(type => type.value > 0)
      .sort((a, b) => b.value - a.value);
  };
  
  // Get consumption trend by type
  const getConsumptionByTypeData = () => {
    if (!masterData || !typeBreakdown) return [];
    
    const months = ['Jan-25', 'Feb-25', 'Mar-25'];
    
    if (selectedType === 'all') {
      // Return total consumption for major types by month
      return months.map(month => {
        const data = { name: month.replace('-25', '') };
        
        // Limit to top 5 types by consumption
        const topTypes = [...typeBreakdown]
          .sort((a, b) => b.value - a.value)
          .slice(0, 5);
          
        topTypes.forEach(type => {
          const consumption = masterData
            .filter(m => m.Type === type.name && (m.Label === 'L3' || m.Label === 'DC') && m['Acct #'] !== '4300322')
            .reduce((sum, meter) => sum + (meter[month] || 0), 0);
            
          data[type.name] = consumption;
        });
        
        return data;
      });
    } else {
      // Return consumption for the selected type by month
      return months.map(month => {
        const consumption = masterData
          .filter(m => m.Type === selectedType && (m.Label === 'L3' || m.Label === 'DC') && m['Acct #'] !== '4300322')
          .reduce((sum, meter) => sum + (meter[month] || 0), 0);
          
        return {
          name: month.replace('-25', ''),
          value: consumption
        };
      });
    }
  };
  
  // Loading state
  if (isLoading) {
    return (
      <div className="flex items-center justify-center h-screen bg-gray-50">
        <div className="text-center">
          <div className="inline-block animate-spin rounded-full h-8 w-8 border-t-2 border-b-2 border-blue-500 mb-4"></div>
          <p className="text-gray-600">Loading data...</p>
        </div>
      </div>
    );
  }
  
  return (
    <div className="min-h-screen bg-gray-50">
      {/* Header with breadcrumb */}
      <header className="bg-white border-b border-gray-200">
        <div className="container mx-auto px-4 py-2">
          <div className="flex items-center text-sm text-gray-500">
            <Home size={16} className="mr-2" />
            <ChevronRight size={14} className="mx-1" />
            <span>Water System</span>
          </div>
        </div>
      </header>
      
      {/* Title section */}
      <div className="bg-white border-b border-gray-200 shadow-sm">
        <div className="container mx-auto px-6 py-5">
          <div className="flex items-center">
            <div className="h-10 w-10 rounded-full bg-blue-100 flex items-center justify-center mr-3">
              <Droplet size={20} className="text-blue-600" />
            </div>
            <div>
              <h1 className="text-xl font-semibold text-gray-800">Water System</h1>
              <p className="text-sm text-gray-500">Management and monitoring of water distribution across Muscat Bay</p>
            </div>
          </div>
        </div>
      </div>
      
      {/* Main dashboard area */}
      <div className="container mx-auto px-4 py-5">
        {/* Dashboard header */}
        <div className="bg-gray-800 rounded-lg shadow-sm overflow-hidden mb-5" style={{ backgroundColor: COLORS.header }}>
          <div className="px-6 py-4 flex justify-between items-center">
            <div>
              <h2 className="text-lg font-medium text-white">Muscat Bay Water Monitoring</h2>
              <p className="text-gray-400 text-sm">Dashboard overview of water consumption and losses</p>
            </div>
            <div>
              <Dropdown 
                value="Total (Q1 2025)"
                options={[
                  { value: 'Total (Q1 2025)', label: 'Total (Q1 2025)' },
                  { value: 'Jan-25', label: 'January 2025' },
                  { value: 'Feb-25', label: 'February 2025' },
                  { value: 'Mar-25', label: 'March 2025' }
                ]}
                onChange={(value) => {
                  if (value !== 'Total (Q1 2025)') {
                    setSelectedMonth(value);
                  }
                }}
                className="text-white bg-opacity-50"
              />
            </div>
          </div>
        </div>
        
        {/* Tabs Navigation */}
        <div className="border-b border-gray-200 mb-6">
          <nav className="flex -mb-px">
            <TabButton 
              active={activeTab === 'overview'} 
              onClick={() => setActiveTab('overview')}
            >
              Overview
            </TabButton>
            <TabButton 
              active={activeTab === 'zone'} 
              onClick={() => setActiveTab('zone')}
            >
              Zone Analysis
            </TabButton>
            <TabButton 
              active={activeTab === 'consumption'} 
              onClick={() => setActiveTab('consumption')}
            >
              Consumption Patterns
            </TabButton>
            <TabButton 
              active={activeTab === 'loss'} 
              onClick={() => setActiveTab('loss')}
            >
              Loss Analysis
            </TabButton>
          </nav>
        </div>
        
        {/* Overview Tab Content */}
        {activeTab === 'overview' && metrics && (
          <div>
            {/* Overview Stats */}
            <div className="mb-8">
              <h2 className="text-lg font-semibold text-gray-900 mb-4">Key Performance Indicators</h2>
              <div className="grid grid-cols-1 md:grid-cols-3 lg:grid-cols-5 gap-4">
                <Card>
                  <div className="p-4 flex items-start">
                    <div className="mr-3 w-8 h-8 rounded-full bg-blue-100 flex items-center justify-center">
                      <Droplet size={18} className="text-blue-600" />
                    </div>
                    <div>
                      <p className="text-sm font-medium text-gray-500">Total Supply</p>
                      <p className="text-xl font-bold text-gray-900">{formatNumber(metrics.l1Supply)} m³</p>
                    </div>
                  </div>
                </Card>
                
                <Card>
                  <div className="p-4 flex items-start">
                    <div className="mr-3 w-8 h-8 rounded-full bg-teal-100 flex items-center justify-center">
                      <BarChart2 size={18} className="text-teal-600" />
                    </div>
                    <div>
                      <p className="text-sm font-medium text-gray-500">Total Consumption</p>
                      <p className="text-xl font-bold text-gray-900">{formatNumber(metrics.l3Volume)} m³</p>
                    </div>
                  </div>
                </Card>
                
                <Card>
                  <div className="p-4 flex items-start">
                    <div className="mr-3 w-8 h-8 rounded-full bg-purple-100 flex items-center justify-center">
                      <Activity size={18} className="text-purple-600" />
                    </div>
                    <div>
                      <p className="text-sm font-medium text-gray-500">Total Loss (NRW)</p>
                      <p className="text-xl font-bold text-gray-900">{formatNumber(metrics.totalLossPercent, 1)}%</p>
                      <p className="text-xs text-gray-500">{formatNumber(metrics.totalLoss)} m³</p>
                    </div>
                  </div>
                </Card>
                
                <Card>
                  <div className="p-4 flex items-start">
                    <div className="mr-3 w-8 h-8 rounded-full bg-red-100 flex items-center justify-center">
                      <TrendingDown size={18} className="text-red-600" />
                    </div>
                    <div>
                      <p className="text-sm font-medium text-gray-500">Stage 1 Loss</p>
                      <p className="text-xl font-bold text-gray-900">{formatNumber(metrics.stage1LossPercent, 1)}%</p>
                      <p className="text-xs text-gray-500">{formatNumber(metrics.stage1Loss)} m³</p>
                    </div>
                  </div>
                </Card>
                
                <Card>
                  <div className="p-4 flex items-start">
                    <div className="mr-3 w-8 h-8 rounded-full bg-amber-100 flex items-center justify-center">
                      <TrendingDown size={18} className="text-amber-600" />
                    </div>
                    <div>
                      <p className="text-sm font-medium text-gray-500">Stage 2 Loss</p>
                      <p className="text-xl font-bold text-gray-900">{formatNumber(metrics.stage2LossPercent, 1)}%</p>
                      <p className="text-xs text-gray-500">{formatNumber(metrics.stage2Loss)} m³</p>
                    </div>
                  </div>
                </Card>
              </div>
            </div>
            
            {/* Trend Chart */}
            <Card className="p-6 mb-8">
              <h3 className="text-base font-semibold text-gray-900 mb-6">Water Consumption & Loss Trends (2025)</h3>
              <div className="h-72">
                <ResponsiveContainer width="100%" height="100%">
                  <ComposedChart
                    data={getTrendData()}
                    margin={{ top: 10, right: 30, left: 10, bottom: 10 }}
                  >
                    <defs>
                      <linearGradient id="colorSupply" x1="0" y1="0" x2="0" y2="1">
                        <stop offset="5%" stopColor={COLORS.primary} stopOpacity={0.2}/>
                        <stop offset="95%" stopColor={COLORS.primary} stopOpacity={0}/>
                      </linearGradient>
                      <linearGradient id="colorConsumption" x1="0" y1="0" x2="0" y2="1">
                        <stop offset="5%" stopColor={COLORS.secondary} stopOpacity={0.2}/>
                        <stop offset="95%" stopColor={COLORS.secondary} stopOpacity={0}/>
                      </linearGradient>
                    </defs>
                    <CartesianGrid strokeDasharray="3 3" vertical={false} stroke="#eee" />
                    <XAxis dataKey="name" 
                      axisLine={false} 
                      tickLine={false}
                      fontSize={12}
                      tickMargin={10}
                    />
                    <YAxis 
                      yAxisId="left"
                      axisLine={false} 
                      tickLine={false}
                      fontSize={12}
                      tickMargin={10}
                    />
                    <YAxis 
                      yAxisId="right" 
                      orientation="right" 
                      axisLine={false} 
                      tickLine={false}
                      fontSize={12}
                      tickMargin={10}
                      unit="%"
                    />
                    <Tooltip contentStyle={{ borderRadius: '4px' }} />
                    <Legend align="right" verticalAlign="top" iconType="circle" />
                    <Area 
                      yAxisId="left"
                      type="monotone"
                      dataKey="l1Supply" 
                      name="Supply (m³)" 
                      stroke={COLORS.primary}
                      strokeWidth={2}
                      fill="url(#colorSupply)"
                      activeDot={{ r: 6, strokeWidth: 0 }}
                    />
                    <Area 
                      yAxisId="left"
                      type="monotone"
                      dataKey="l3Consumption" 
                      name="Consumption (m³)" 
                      stroke={COLORS.secondary}
                      strokeWidth={2}
                      fill="url(#colorConsumption)"
                      activeDot={{ r: 6, strokeWidth: 0 }}
                    />
                    <Line 
                      yAxisId="right"
                      type="monotone" 
                      dataKey="lossPercent" 
                      name="Loss (%)" 
                      stroke={COLORS.danger}
                      strokeWidth={3}
                      dot={{ r: 6, fill: COLORS.danger, strokeWidth: 0 }}
                      activeDot={{ r: 8, strokeWidth: 0 }}
                    />
                  </ComposedChart>
                </ResponsiveContainer>
              </div>
            </Card>
            
            {/* Charts Section */}
            <div className="grid grid-cols-1 lg:grid-cols-2 gap-6 mb-8">
              {/* Consumption Breakdown Chart */}
              <Card className="p-6">
                <h3 className="text-base font-semibold text-gray-900 mb-6">Consumption Breakdown by Type</h3>
                <div className="h-64">
                  <ResponsiveContainer width="100%" height="100%">
                    <PieChart>
                      <Pie
                        data={typeBreakdown}
                        cx="50%"
                        cy="50%"
                        innerRadius={60}
                        outerRadius={90}
                        paddingAngle={2}
                        fill="#8884d8"
                        dataKey="value"
                      >
                        {typeBreakdown.map((entry, index) => (
                          <Cell key={`cell-${index}`} fill={COLORS.chart[index % COLORS.chart.length]} />
                        ))}
                      </Pie>
                      <Tooltip 
                        formatter={(value) => [formatNumber(value) + ' m³', 'Consumption']}
                        contentStyle={{ borderRadius: '4px' }} 
                      />
                      <Legend 
                        layout="vertical" 
                        verticalAlign="middle" 
                        align="right"
                        iconType="circle"
                      />
                    </PieChart>
                  </ResponsiveContainer>
                </div>
              </Card>
              
              {/* Top Loss Zones Chart */}
              <Card className="p-6">
                <h3 className="text-base font-semibold text-gray-900 mb-6">Top Loss Zones</h3>
                <div className="h-64">
                  <ResponsiveContainer width="100%" height="100%">
                    <BarChart 
                      data={getTopLossZones()}
                      layout="vertical"
                      margin={{ top: 5, right: 30, left: 20, bottom: 5 }}
                    >
                      <CartesianGrid strokeDasharray="3 3" horizontal={true} vertical={false} />
                      <XAxis type="number" axisLine={false} tickLine={false} unit="%" />
                      <YAxis 
                        dataKey="name" 
                        type="category" 
                        axisLine={false} 
                        tickLine={false}
                        width={75}
                        fontSize={12}
                      />
                      <Tooltip 
                        formatter={(value, name) => [`${value}%`, 'Loss']}
                        contentStyle={{ borderRadius: '4px' }}
                      />
                      {/* Gradient for bar chart */}
                      <defs>
                        <linearGradient id="barGradient" x1="0" y1="0" x2="1" y2="0">
                          <stop offset="0%" stopColor={COLORS.danger} stopOpacity={0.7} />
                          <stop offset="100%" stopColor={COLORS.danger} stopOpacity={1} />
                        </linearGradient>
                      </defs>
                      <Bar 
                        dataKey="lossPercent" 
                        fill="url(#barGradient)" 
                        name="Loss %" 
                        radius={[0, 4, 4, 0]}
                        barSize={16}
                      />
                    </BarChart>
                  </ResponsiveContainer>
                </div>
              </Card>
            </div>
          </div>
        )}
        
        {/* Zone Analysis Tab Content */}
        {activeTab === 'zone' && metrics && selectedZone && zoneMetrics[selectedZone] && (
          <Card className="overflow-hidden">
            {/* Zone details header */}
            <div className="px-6 py-4 flex items-center justify-between border-b border-gray-200">
              <h3 className="text-lg font-medium text-gray-800">Zone Details for Total</h3>
              <Dropdown 
                value={selectedZone}
                options={Object.keys(zoneMetrics).map(zone => ({
                  value: zone,
                  label: zone.replace('_', ' ')
                }))}
                onChange={setSelectedZone}
              />
            </div>
            
            {/* Circular gauges arranged in 2x2 grid, matching screenshot exactly */}
            <div className="px-8 py-10 border-b border-gray-200">
              <div className="grid grid-cols-2 gap-x-16 gap-y-12">
                {/* Top row */}
                <div className="flex flex-col items-center">
                  <CircularGauge 
                    value={zoneMetrics[selectedZone].bulkReading}
                    total={zoneMetrics[selectedZone].bulkReading * 1.5}
                    color={COLORS.primary}
                    label="L2 Bulk Meter Reading"
                  />
                </div>
                
                <div className="flex flex-col items-center">
                  <CircularGauge 
                    value={zoneMetrics[selectedZone].l3Sum}
                    total={zoneMetrics[selectedZone].bulkReading * 1.5}
                    color={COLORS.secondary}
                    label="Sum of L3 Consumption"
                  />
                </div>
                
                {/* Bottom row - switched positions to match screenshot */}
                <div className="flex flex-col items-center">
                  <CircularGauge 
                    value={0}
                    total={100}
                    overrideValue={zoneMetrics[selectedZone].lossPercent}
                    color={COLORS.lossPercentage}
                    label="Loss Percentage"
                    isPercentage={true}
                  />
                </div>
                
                <div className="flex flex-col items-center">
                  <CircularGauge 
                    value={zoneMetrics[selectedZone].loss}
                    total={zoneMetrics[selectedZone].bulkReading}
                    color={COLORS.lossVolume}
                    label="Loss Volume"
                  />
                </div>
              </div>
            </div>
            
            {/* Search and table */}
            <div className="p-6">
              <div className="mb-5">
                <h4 className="text-sm font-medium text-gray-700 mb-2">Search Properties</h4>
                <div className="relative">
                  <div className="absolute inset-y-0 left-0 pl-3 flex items-center pointer-events-none">
                    <Search size={16} className="text-gray-400" />
                  </div>
                  <input
                    type="text"
                    className="block w-full pl-10 pr-3 py-2 border border-gray-300 rounded-md leading-5 bg-white placeholder-gray-500 focus:outline-none focus:placeholder-gray-400 focus:ring-1 focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                    placeholder="Search by meter label, account#, or type..."
                    value={searchTerm}
                    onChange={(e) => setSearchTerm(e.target.value)}
                  />
                </div>
              </div>
              
              {/* Meters table */}
              <div className="overflow-x-auto">
                <table className="min-w-full divide-y divide-gray-200">
                  <thead>
                    <tr>
                      <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                        <div className="flex items-center cursor-pointer">
                          Account #
                          <ArrowUpDown size={14} className="ml-1 text-gray-400" />
                        </div>
                      </th>
                      <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                        <div className="flex items-center cursor-pointer">
                          Meter Label
                          <ArrowUpDown size={14} className="ml-1 text-gray-400" />
                        </div>
                      </th>
                      <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                        <div className="flex items-center cursor-pointer">
                          Parent Meter
                          <ArrowUpDown size={14} className="ml-1 text-gray-400" />
                        </div>
                      </th>
                      <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                        <div className="flex items-center cursor-pointer">
                          Type
                          <ArrowUpDown size={14} className="ml-1 text-gray-400" />
                        </div>
                      </th>
                      <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                        <div className="flex items-center cursor-pointer">
                          Consumption
                          <ArrowUpDown size={14} className="ml-1 text-gray-400" />
                        </div>
                      </th>
                    </tr>
                  </thead>
                  <tbody className="bg-white divide-y divide-gray-200">
                    {getFilteredMeters().map((meter, index) => (
                      <tr key={index} className="hover:bg-gray-50 transition-colors">
                        <td className="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">
                          {meter.accountNumber}
                        </td>
                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                          {meter.meterLabel}
                        </td>
                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                          {`ZONE ${selectedZone.replace('Zone_', '')} ( BULK ZONE ${selectedZone.replace('Zone_', '')} )`}
                        </td>
                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                          {meter.type}
                        </td>
                        <td className="px-6 py-4 whitespace-nowrap">
                          <ConsumptionBar 
                            value={meter.consumption}
                            maxValue={getMaxConsumption()}
                          />
                        </td>
                      </tr>
                    ))}
                    {getFilteredMeters().length === 0 && (
                      <tr>
                        <td colSpan="5" className="px-6 py-4 text-center text-sm text-gray-500">
                          No meters found matching your search criteria.
                        </td>
                      </tr>
                    )}
                  </tbody>
                </table>
              </div>
            </div>
            
            {/* Analysis Notes */}
            <div className="px-6 py-4 bg-gray-50 border-t border-gray-200">
              <h4 className="text-sm font-medium text-gray-700 mb-2">Analysis Notes</h4>
              <p className="text-sm text-gray-600">
                {zoneMetrics[selectedZone].isNegativeLoss ? 
                  `This zone shows a small surplus, with L3 consumption slightly exceeding L2 bulk readings. This is generally within acceptable metering tolerances.` :
                  `This zone shows a ${zoneMetrics[selectedZone].lossPercent > 10 ? 'significant' : 'small'} loss of ${formatNumber(zoneMetrics[selectedZone].loss)} m³ (${formatNumber(zoneMetrics[selectedZone].lossPercent, 1)}%). ${zoneMetrics[selectedZone].lossPercent > 20 ? 'This requires investigation for potential leaks or meter issues.' : 'This is within normal operational parameters.'}`
                }
              </p>
            </div>
          </Card>
        )}
        
        {/* Consumption Patterns Tab Content */}
        {activeTab === 'consumption' && metrics && (
          <div>
            <div className="flex items-center justify-between mb-5">
              <h2 className="text-lg font-semibold text-gray-900">Consumption by Type</h2>
              
              <Dropdown 
                value={selectedType}
                options={[
                  { value: 'all', label: 'All Types' },
                  ...typeBreakdown.map(type => ({
                    value: type.name, 
                    label: type.name
                  }))
                ]}
                onChange={setSelectedType}
              />
            </div>
            
            {/* Type Summary Cards */}
            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4 mb-6">
              {getTypeBreakdownData().slice(0, 4).map((type, index) => (
                <Card key={index} className={`p-4 ${selectedType === type.name ? 'ring-2 ring-blue-500' : ''}`} onClick={() => setSelectedType(type.name)}>
                  <div className="flex items-start">
                    <div className="mr-3 h-10 w-10 rounded-full flex items-center justify-center" style={{ backgroundColor: `${COLORS.chart[index]}20` }}>
                      <div className="h-6 w-6 rounded-full" style={{ backgroundColor: COLORS.chart[index] }}></div>
                    </div>
                    <div>
                      <h3 className="font-medium text-gray-900">{type.name}</h3>
                      <p className="text-2xl font-bold text-gray-800 mt-1">{formatNumber(type.value)} m³</p>
                      <p className="text-xs text-gray-500 mt-1">
                        {((type.value / metrics.l3Volume) * 100).toFixed(1)}% of total
                      </p>
                    </div>
                  </div>
                </Card>
              ))}
            </div>
            
            {/* Charts Section */}
            <div className="grid grid-cols-1 lg:grid-cols-2 gap-6 mb-8">
              {/* Type Distribution Pie Chart */}
              <Card className="p-6">
                <h3 className="text-base font-semibold text-gray-900 mb-6">Consumption Distribution</h3>
                <div className="h-64">
                  <ResponsiveContainer width="100%" height="100%">
                    <PieChart>
                      <Pie
                        data={getTypeBreakdownData()}
                        cx="50%"
                        cy="50%"
                        innerRadius={60}
                        outerRadius={90}
                        paddingAngle={2}
                        dataKey="value"
                      >
                        {getTypeBreakdownData().map((entry, index) => (
                          <Cell 
                            key={`cell-${index}`} 
                            fill={COLORS.chart[index % COLORS.chart.length]} 
                            opacity={selectedType === 'all' || selectedType === entry.name ? 1 : 0.4}
                          />
                        ))}
                      </Pie>
                      <Tooltip 
                        formatter={(value) => [formatNumber(value) + ' m³', 'Consumption']}
                        contentStyle={{ borderRadius: '4px' }} 
                      />
                      <Legend 
                        layout="vertical" 
                        verticalAlign="middle" 
                        align="right"
                        iconType="circle"
                        formatter={(value, entry, index) => (
                          <span style={{ 
                            color: selectedType === 'all' || selectedType === value ? '#333' : '#999',
                            fontWeight: selectedType === value ? 'bold' : 'normal'
                          }}>
                            {value}
                          </span>
                        )}
                        onClick={(data) => setSelectedType(data.value)}
                      />
                    </PieChart>
                  </ResponsiveContainer>
                </div>
              </Card>
              
              {/* Consumption Trend Chart */}
              <Card className="p-6">
                <h3 className="text-base font-semibold text-gray-900 mb-6">
                  {selectedType === 'all' ? 'Consumption Trends by Type' : `${selectedType} Consumption Trend`}
                </h3>
                <div className="h-64">
                  <ResponsiveContainer width="100%" height="100%">
                    {selectedType === 'all' ? (
                      <BarChart
                        data={getConsumptionByTypeData()}
                        margin={{ top: 20, right: 30, left: 20, bottom: 5 }}
                      >
                        <CartesianGrid strokeDasharray="3 3" vertical={false} />
                        <XAxis dataKey="name" axisLine={false} tickLine={false} />
                        <YAxis axisLine={false} tickLine={false} />
                        <Tooltip />
                        <Legend />
                        {getTypeBreakdownData().slice(0, 5).map((type, index) => (
                          <Bar 
                            key={`bar-${index}`}
                            dataKey={type.name} 
                            stackId="a" 
                            fill={COLORS.chart[index % COLORS.chart.length]} 
                          />
                        ))}
                      </BarChart>
                    ) : (
                      <AreaChart
                        data={getConsumptionByTypeData()}
                        margin={{ top: 20, right: 30, left: 20, bottom: 5 }}
                      >
                        <defs>
                          <linearGradient id="typeGradient" x1="0" y1="0" x2="0" y2="1">
                            <stop offset="5%" stopColor={COLORS.chart[getTypeBreakdownData().findIndex(t => t.name === selectedType) % COLORS.chart.length]} stopOpacity={0.3}/>
                            <stop offset="95%" stopColor={COLORS.chart[getTypeBreakdownData().findIndex(t => t.name === selectedType) % COLORS.chart.length]} stopOpacity={0}/>
                          </linearGradient>
                        </defs>
                        <CartesianGrid strokeDasharray="3 3" vertical={false} />
                        <XAxis dataKey="name" axisLine={false} tickLine={false} />
                        <YAxis axisLine={false} tickLine={false} />
                        <Tooltip formatter={(value) => [formatNumber(value) + ' m³', 'Consumption']} />
                        <Area 
                          type="monotone" 
                          dataKey="value" 
                          name={selectedType} 
                          stroke={COLORS.chart[getTypeBreakdownData().findIndex(t => t.name === selectedType) % COLORS.chart.length]} 
                          fillOpacity={1}
                          fill="url(#typeGradient)"
                        />
                      </AreaChart>
                    )}
                  </ResponsiveContainer>
                </div>
              </Card>
            </div>
            
            {/* Usage Analysis */}
            <Card className="p-6">
              <h3 className="text-base font-semibold text-gray-900 mb-4">Usage Analysis</h3>
              
              <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
                <div>
                  <h4 className="text-sm font-medium text-gray-700 mb-2">Residential Usage</h4>
                  <div className="space-y-2">
                    {getTypeBreakdownData()
                      .filter(t => t.name.includes('Residential'))
                      .map((type, index) => (
                        <div key={index} className="flex justify-between items-center p-3 bg-gray-50 rounded-lg">
                          <span className="text-sm text-gray-700">{type.name}:</span>
                          <span className="text-sm font-medium text-gray-900">{formatNumber(type.value)} m³</span>
                        </div>
                    ))}
                  </div>
                </div>
                
                <div>
                  <h4 className="text-sm font-medium text-gray-700 mb-2">Commercial Usage</h4>
                  <div className="space-y-2">
                    {getTypeBreakdownData()
                      .filter(t => t.name.includes('Retail') || t.name.includes('Building'))
                      .map((type, index) => (
                        <div key={index} className="flex justify-between items-center p-3 bg-gray-50 rounded-lg">
                          <span className="text-sm text-gray-700">{type.name}:</span>
                          <span className="text-sm font-medium text-gray-900">{formatNumber(type.value)} m³</span>
                        </div>
                    ))}
                  </div>
                </div>
                
                <div>
                  <h4 className="text-sm font-medium text-gray-700 mb-2">Other Usage</h4>
                  <div className="space-y-2">
                    {getTypeBreakdownData()
                      .filter(t => !t.name.includes('Residential') && !t.name.includes('Retail') && !t.name.includes('Building'))
                      .map((type, index) => (
                        <div key={index} className="flex justify-between items-center p-3 bg-gray-50 rounded-lg">
                          <span className="text-sm text-gray-700">{type.name}:</span>
                          <span className="text-sm font-medium text-gray-900">{formatNumber(type.value)} m³</span>
                        </div>
                    ))}
                  </div>
                </div>
              </div>
            </Card>
          </div>
        )}
        
        {/* Loss Analysis Tab Content */}
        {metrics && activeTab === 'loss' && (
          <div>
            <h2 className="text-lg font-semibold text-gray-900 mb-6">Loss Analysis</h2>
            
            {/* Stage Loss Summary */}
            <div className="mb-6 grid grid-cols-1 md:grid-cols-2 gap-6">
              {/* Stage 1 Loss Analysis */}
              <Card className="overflow-hidden">
                <div className="px-6 py-4 bg-gray-50 border-b border-gray-200">
                  <div className="flex items-center">
                    <TrendingDown size={16} className="text-blue-600 mr-2" />
                    <h3 className="text-base font-medium text-gray-900">Stage 1 Loss Analysis</h3>
                  </div>
                </div>
                <div className="p-6">
                  <div className="space-y-4">
                    <div className="flex justify-between border-b border-gray-100 pb-3">
                      <span className="text-sm font-medium text-gray-700">L1 Main Bulk Supply:</span>
                      <span className="text-sm font-semibold text-gray-900">{formatNumber(metrics.l1Supply)} m³</span>
                    </div>
                    <div className="flex justify-between border-b border-gray-100 pb-3">
                      <span className="text-sm font-medium text-gray-700">L2 Zone Bulk + DC:</span>
                      <span className="text-sm font-semibold text-gray-900">{formatNumber(metrics.l2Volume)} m³</span>
                    </div>
                    <div className="flex justify-between">
                      <span className="text-sm font-medium text-gray-700">Stage 1 Loss:</span>
                      <span className={`text-sm font-semibold ${metrics.stage1Loss < 0 ? 'text-green-600' : 'text-red-600'}`}>
                        {formatNumber(metrics.stage1Loss)} m³ ({formatNumber(metrics.stage1LossPercent, 1)}%)
                      </span>
                    </div>
                  </div>
                  
                  <div className="mt-6 p-4 bg-gray-50 rounded-lg border border-gray-200">
                    <div className="flex items-start">
                      <div className="mt-0.5 mr-2 flex-shrink-0">
                        <FileText size={14} className="text-gray-500" />
                      </div>
                      <div>
                        <h4 className="text-xs font-semibold text-gray-700 uppercase tracking-wide mb-1">Analysis</h4>
                        <p className="text-sm text-gray-600">
                          {metrics.stage1Loss < 0 ? 
                            "The negative Stage 1 Loss indicates more water is measured at L2 points than at the source. This may be due to metering inaccuracies, timing differences in readings, or calibration issues with the main bulk meter." :
                            "Stage 1 Loss represents water lost between the main source (L1) and primary distribution points (L2). This could be due to leaks in the trunk main, unauthorized connections, or meter inaccuracies."
                          }
                        </p>
                      </div>
                    </div>
                  </div>
                </div>
              </Card>
              
              {/* Stage 2 Loss Analysis */}
              <Card className="overflow-hidden">
                <div className="px-6 py-4 bg-gray-50 border-b border-gray-200">
                  <div className="flex items-center">
                    <TrendingDown size={16} className="text-purple-600 mr-2" />
                    <h3 className="text-base font-medium text-gray-900">Stage 2 Loss Analysis</h3>
                  </div>
                </div>
                <div className="p-6">
                  <div className="space-y-4">
                    <div className="flex justify-between border-b border-gray-100 pb-3">
                      <span className="text-sm font-medium text-gray-700">L2 Zone Bulk + DC:</span>
                      <span className="text-sm font-semibold text-gray-900">{formatNumber(metrics.l2Volume)} m³</span>
                    </div>
                    <div className="flex justify-between border-b border-gray-100 pb-3">
                      <span className="text-sm font-medium text-gray-700">L3 Individual + DC:</span>
                      <span className="text-sm font-semibold text-gray-900">{formatNumber(metrics.l3Volume)} m³</span>
                    </div>
                    <div className="flex justify-between">
                      <span className="text-sm font-medium text-gray-700">Stage 2 Loss:</span>
                      <span className="text-sm font-semibold text-red-600">
                        {formatNumber(metrics.stage2Loss)} m³ ({formatNumber(metrics.stage2LossPercent, 1)}%)
                      </span>
                    </div>
                  </div>
                  
                  <div className="mt-6 p-4 bg-gray-50 rounded-lg border border-gray-200">
                    <div className="flex items-start">
                      <div className="mt-0.5 mr-2 flex-shrink-0">
                        <FileText size={14} className="text-gray-500" />
                      </div>
                      <div>
                        <h4 className="text-xs font-semibold text-gray-700 uppercase tracking-wide mb-1">Analysis</h4>
                        <p className="text-sm text-gray-600">
                          Stage 2 Loss represents water lost within distribution networks after L2 measurement points. This includes leaks in distribution pipes, meter inaccuracies, and potentially unauthorized usage within zones. Zone 05 and Zone 03(A) have the highest internal losses and should be prioritized for investigation.
                        </p>
                      </div>
                    </div>
                  </div>
                </div>
              </Card>
            </div>
            
            {/* Zone Loss Comparison Table */}
            <Card className="overflow-hidden mb-8">
              <div className="px-6 py-4 bg-gray-50 border-b border-gray-200">
                <h3 className="text-base font-medium text-gray-900">Internal Zone Loss Comparison</h3>
              </div>
              <div className="overflow-x-auto">
                <table className="min-w-full divide-y divide-gray-200">
                  <thead className="bg-gray-50">
                    <tr>
                      <th scope="col" className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                        Zone
                      </th>
                      <th scope="col" className="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">
                        Bulk Reading (m³)
                      </th>
                      <th scope="col" className="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">
                        L3 Consumption (m³)
                      </th>
                      <th scope="col" className="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">
                        Loss (m³)
                      </th>
                      <th scope="col" className="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">
                        Loss (%)
                      </th>
                      <th scope="col" className="px-6 py-3 text-center text-xs font-medium text-gray-500 uppercase tracking-wider">
                        Status
                      </th>
                    </tr>
                  </thead>
                  <tbody className="bg-white divide-y divide-gray-200">
                    {Object.values(zoneMetrics)
                      .sort((a, b) => b.lossPercent - a.lossPercent)
                      .map((zone, index) => (
                        <tr key={index} className="hover:bg-gray-50 transition-colors">
                          <td className="px-6 py-3 text-sm font-medium text-gray-900">
                            {zone.zoneName.replace('_', ' ')}
                          </td>
                          <td className="px-6 py-3 text-sm text-gray-900 text-right">
                            {formatNumber(zone.bulkReading)}
                          </td>
                          <td className="px-6 py-3 text-sm text-gray-900 text-right">
                            {formatNumber(zone.l3Sum)}
                          </td>
                          <td className="px-6 py-3 text-sm text-gray-900 text-right">
                            {formatNumber(zone.loss)}
                          </td>
                          <td className={`px-6 py-3 text-sm font-medium text-right ${
                            zone.lossPercent > 50 ? 'text-red-600' : 
                            zone.lossPercent > 20 ? 'text-amber-600' : 
                            zone.isNegativeLoss ? 'text-green-600' : 'text-gray-900'
                          }`}>
                            {formatNumber(zone.lossPercent, 1)}%
                          </td>
                          <td className="px-6 py-3 text-sm text-center">
                            <span className={`px-2 py-1 inline-flex text-xs leading-5 font-semibold rounded-full ${
                              zone.lossPercent > 50 ? 'bg-red-100 text-red-800' : 
                              zone.lossPercent > 20 ? 'bg-amber-100 text-amber-800' : 
                              zone.isNegativeLoss ? 'bg-green-100 text-green-800' : 'bg-blue-100 text-blue-800'
                            }`}>
                              {zone.lossPercent > 50 ? 'Critical' : 
                               zone.lossPercent > 20 ? 'Warning' : 
                               zone.isNegativeLoss ? 'Surplus' : 'Normal'}
                            </span>
                          </td>
                        </tr>
                      ))}
                  </tbody>
                </table>
              </div>
            </Card>
            
            {/* Zone Comparison Chart */}
            <Card className="p-6 mb-6">
              <h3 className="text-base font-semibold text-gray-900 mb-4">Zone Loss Comparison</h3>
              <div className="h-80">
                <ResponsiveContainer width="100%" height="100%">
                  <BarChart
                    data={Object.values(zoneMetrics)
                      .sort((a, b) => b.lossPercent - a.lossPercent)
                      .map(zone => ({
                        name: zone.zoneName.replace('Zone_', ''),
                        bulkReading: zone.bulkReading,
                        consumption: zone.l3Sum,
                        loss: zone.loss,
                        lossPercent: parseFloat(zone.lossPercent.toFixed(1))
                      }))}
                    margin={{ top: 20, right: 30, left: 20, bottom: 5 }}
                  >
                    <CartesianGrid strokeDasharray="3 3" />
                    <XAxis dataKey="name" />
                    <YAxis yAxisId="left" orientation="left" />
                    <YAxis yAxisId="right" orientation="right" unit="%" />
                    <Tooltip contentStyle={{ borderRadius: '4px' }} />
                    <Legend />
                    <defs>
                      <linearGradient id="colorBulk" x1="0" y1="0" x2="0" y2="1">
                        <stop offset="5%" stopColor={COLORS.primary} stopOpacity={0.8}/>
                        <stop offset="95%" stopColor={COLORS.primary} stopOpacity={0.4}/>
                      </linearGradient>
                      <linearGradient id="colorConsumption" x1="0" y1="0" x2="0" y2="1">
                        <stop offset="5%" stopColor={COLORS.secondary} stopOpacity={0.8}/>
                        <stop offset="95%" stopColor={COLORS.secondary} stopOpacity={0.4}/>
                      </linearGradient>
                    </defs>
                    <Bar yAxisId="left" dataKey="bulkReading" name="Bulk Reading" fill="url(#colorBulk)" radius={[4, 4, 0, 0]} />
                    <Bar yAxisId="left" dataKey="consumption" name="Consumption" fill="url(#colorConsumption)" radius={[4, 4, 0, 0]} />
                    <Line yAxisId="right" type="monotone" dataKey="lossPercent" name="Loss %" stroke={COLORS.danger} strokeWidth={3} />
                  </BarChart>
                </ResponsiveContainer>
              </div>
            </Card>
          </div>
        )}
      </div>
    </div>
  );
};

export default DashboardPage;
